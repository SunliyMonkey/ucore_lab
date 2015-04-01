#report



---

# Lab1 report

## [练习1]

**[练习1.1] 操作系统镜像文件ucore.img是如何一步一步生成的?(需要比较详细地解释 Makefile中每一条相关命令和命令参数的含义,以及说明命令导致的结果)**

**UCOREIMG分析**
在Makefile查找ucore.img,可以看见如下的配置，结合目录bin下文件，做以下注释。


```
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

#UCOREIMG与kernel，bootblock相关
$(UCOREIMG): $(kernel) $(bootblock)
    
    #初始化ucore.img的大小，设置为10000个扇区
	$(V)dd if=/dev/zero of=$@ count=10000 
	#用bootblock(引导程序)初始磁盘主引导扇区
	$(V)dd if=$(bootblock) of=$@ conv=notrunc 
	#用kernel(内核镜像)初始ucore.img第一个扇区之后的区域
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```

PS： 变量与大小信息
```
UCOREIMG=bin/ucore.img
kernel=bin/kernel
bootblock=bin/bootblock
-rw-rw-r-- 1 moocos moocos 5120000  3月 30 15:32 ucore.img
-rwxrwxr-x 1 moocos moocos   74871  3月 30 15:32 kernel
-rw-rw-r-- 1 moocos moocos     512  3月 30 15:09 bootblock
```
**kernel分析**
```
#kernel libs目录下的文件
KOBJS	= $(call read_packet,kernel libs)
# create kernel target
kernel = $(call totarget,kernel)

#已存在
$(kernel): tools/kernel.ld
#与KOBJS中的所有文件相关
$(kernel): $(KOBJS)
	@echo + ld $@
	#利用ld命令，将KOBJS中的文件链接成kernel
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```
其中关于kernel，libs中o文件的生成，由下面的命令控制
```
$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```

附：
1. ld命令：将多个文件链接成目标文件
2. KOBJS信息
```
KOBJS= obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o  obj/libs/printfmt.o obj/libs/string.o
```

**bootblock**分析
```
# create bootblock

#boot目录下的所有文件
bootfiles = $(call listf_cc,boot)
#编译boot目录下的所有文件
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)
#bootblock与bootfiles，sign存在关联
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	#链接成bootblock
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
$(call create_target,bootblock)

# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

**[练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?**
大小512字节，并以0x55，0xAA结尾


## [练习2]使用qemu执行并调试lab1中的软件

**[练习2.1]从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。**
    
配置tools/gdbinit, 执行“make debug”，在调试窗口，输入ni即可单步调试。
```
define hook- stop
    x/i $pc
end
target remote localhost: 1234
```
**[练习2.2]在初始化位置0x7c00设置实地址断点,测试断点正常。**
修改配置文件如下：
```
define hook- stop
    x/i $pc
end
target remote localhost: 1234
b *0x7c00
c
x /10i $pc
```
执行可得到如下结果：
```
Breakpoint 1, 0x00007c00 in ?? ()
=> 0x7c00:      cli    
   0x7c01:      cld    
   0x7c02:      xor    %eax,%eax
   0x7c04:      mov    %eax,%ds
   0x7c06:      mov    %eax,%es
   0x7c08:      mov    %eax,%ss
   0x7c0a:      in     $0x64,%al
   0x7c0c:      test   $0x2,%al
   0x7c0e:      jne    0x7c0a
   0x7c10:      mov    $0xd1,%al
```

**[练习2.3]从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。**

修改Makefile，保存执行的汇编代码
```
debug: $(UCOREIMG)
     $(V)$(QEMU) -S -d in_asm -D asm.log -s -parallel stdio -hda $< -serial null &
     $(V)sleep 2
     $(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
```
配置tools/gdbinit
```
define hook-stop
	x/i $pc
end
target remote :1234
c
```
得到asm.log
```
......
IN:
0x00007c00:  cli
0x00007c01:  cld    
0x00007c02:  xor    %ax,%ax
0x00007c04:  mov    %ax,%ds
0x00007c06:  mov    %ax,%es
0x00007c08:  mov    %ax,%ss

----------------
IN: 
0x00007c0a:  in     $0x64,%al

----------------
IN: 
0x00007c0c:  test   $0x2,%al
0x00007c0e:  jne    0x7c0a

----------------
IN: 
0x00007c10:  mov    $0xd1,%al
0x00007c12:  out    %al,$0x64
0x00007c14:  in     $0x64,%al
0x00007c16:  test   $0x2,%al
0x00007c18:  jne    0x7c14
......
```
比较可以发现与 bootblock.asm，bootasm.S代码一样.


**[练习2.4]自己找一个bootloader或内核中的代码位置，设置断点并进行测试。**
 略
 
 
## [练习3]分析bootloader进入保护模式的过程
**BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。 请分析bootloader是如何完成从实模式进入保护模式的。**

```
#第一步：初始化，关闭中断、方向标志位，清理寄存器
    cli                             
    cld                             
    xorw %ax, %ax
    movw %ax, %ds
    movw %ax, %es
    movw %ax, %ss                   
#第二步：开启A20，通过给键盘控制器发送命令，开启A20
seta20.1:
    inb $0x64, %al   # 读取“读状态寄存器”，等待键盘控制器不繁忙
    testb $0x2, %al  
    jnz seta20.1

    movb $0xd1, %al # 给控制器发送命令'0xd1'，表示修改P2端口
    outb %al, $0x64 

seta20.2:
    inb $0x64, %al # 读取“读状态寄存器”，等待键盘控制器不繁忙
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al # 开启P2端口
    outb %al, $0x60 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1

    #载入GDT表
    lgdt gdtdesc
    
    #将cr0控制寄存器的PE位置1，转换到保护模式
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
```

## [练习4]分析bootloader加载ELF格式的OS的过程

```
    #将磁盘数据的第一页读入到内存
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    //检查ELF正确性
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    //获取program header表起始位置
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    //通过program header的信息，获取目标文件的段内容
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

   #通过程序入口的虚拟地址，进入程序
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```
## [练习5]
[练习5.1]
```
+| 栈底方向 | 高位地址
 | . . .    |
 | . . .    |
 | 参数3    |
 | 参数2    |
 | 参数1    |
 | 返回地址 |
 | 上一层 [ebp] | <- - - - - - - - [ebp]
 | 局部变量 | 低位地址
```
通过以上结构，可以得知：
```
ss:[ebp] 获取上一层调用的基址
ss:[ebp + 1 * 4] - 1 获取上一层函数发生调用的指令位置
ss:[ebp + 2 * 4] 获取参数1
ss:[ebp + 3 * 4] 获取参数2
ss:[ebp + 4 * 4] 获取参数3
```
具体见代码实现
[练习5.2]
```
ebp: 0x00007b08 eip: 0x001009a6 args: 0x00010094 0x00000000 0x00007b38 0x00100092
    kern/debug/kdebug.c:306: print_stackframe+21
ebp: 0x00007b18 eip: 0x00100c95 args: 0x00000000 0x00000000 0x00000000 0x00007b88
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp: 0x00007b38 eip: 0x00100092 args: 0x00000000 0x00007b60 0xffff0000 0x00007b64
    kern/init/init.c:48: grade_backtrace2+33
ebp: 0x00007b58 eip: 0x001000bb args: 0x00000000 0xffff0000 0x00007b84 0x00000029
    kern/init/init.c:53: grade_backtrace1+38
ebp: 0x00007b78 eip: 0x001000d9 args: 0x00000000 0x00100000 0xffff0000 0x0000001d
    kern/init/init.c:58: grade_backtrace0+23
ebp: 0x00007b98 eip: 0x001000fe args: 0x001032fc 0x001032e0 0x0000130a 0x00000000
    kern/init/init.c:63: grade_backtrace+34
ebp: 0x00007bc8 eip: 0x00100055 args: 0x00000000 0x00000000 0x00000000 0x00010094
    kern/init/init.c:28: kern_init+84
ebp: 0x00007bf8 eip: 0x00007d68 args: 0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d67 --
```
最后一行是系统第一次函数bootmain调用的信息，在bootasm.S中有如下调用
```
movl $0x0, %ebp
movl $start, %esp
call bootmain
```
esp = 0x7c00
因此发生堆栈，ebp = esp - 4 = 0x7bf8

## [练习6]
**[练习6.1]中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？**
```
struct gatedesc {
    unsigned gd_off_15_0 : 16;      // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;           // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;           // reserved(should be zero I guess)
    unsigned gd_type : 4;           // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;              // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;              // Present
    unsigned gd_off_31_16 : 16;     // high bits of offset in segment
};
```
表项占用8字节，其中前两个字节和后两个字节拼接在一起表示入口代码位置。


**[练习6.2]请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。**
见代码

**[练习6.3]请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后， 调用print_ticks子程序，向屏幕上打印一行文字”100  ticks”。**
见代码

