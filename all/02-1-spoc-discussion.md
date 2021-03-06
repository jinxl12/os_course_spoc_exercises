#lec 3 SPOC Discussion

##**提前准备**
（请在周一上课前完成）

 - 完成lec3的视频学习和提交对应的在线练习
 - git pull ucore_os_lab, v9_cpu, os_course_spoc_exercises  　in github repos。这样可以在本机上完成课堂练习。
 - 仔细观察自己使用的计算机的启动过程和linux/ucore操作系统运行后的情况。
 - 了解控制流，异常控制流，函数调用,中断，异常(故障)，系统调用（陷阱）,切换，用户态（用户模式），内核态（内核模式）等基本概念。思考一下这些基本概念在linux, ucore, v9-cpu中的os*.c中是如何具体体现的。
 - 思考为什么操作系统需要处理中断，异常，系统调用。这些是必须要有的吗？有哪些好处？有哪些不好的地方？
 - 了解在PC机上有啥中断和异常。搜索“80386　开机　启动”
 - 安装好ucore实验环境，能够编译运行lab8的answer
 - 了解Linux和ucore有哪些系统调用。搜索“linux 系统调用", 搜索lab8中的syscall关键字相关内容。在linux下执行命令: ```man syscalls```
 - 会使用linux中的命令:objdump，nm，file, strace，man, 了解这些命令的用途。
 - 了解如何OS是如何实现中断，异常，或系统调用的。会使用v9-cpu的dis,xc, xem命令（包括启动参数），分析v9-cpu中的os0.c, os2.c，了解与异常，中断，系统调用相关的os设计实现。阅读v9-cpu中的cpu.md文档，了解汇编指令的类型和含义等，了解v9-cpu的细节。
 - 在piazza上就lec3学习中不理解问题进行提问。

## 第三讲 启动、中断、异常和系统调用-思考题

## 3.1 BIOS
 1. 比较UEFI和BIOS的区别。
 
    与legacy BIOS 相比，UEFI最大的几个区别在于：
    1) 编码99%都是由C语言完成；
    2) 一改之前的中断、硬件端口操作的方法，而采用了Driver/protocol的新方式；
    3) 将不支持X86实模式，而直接采用Flat mode（也就是不能用DOS了，现在有些 EFI 或 UEFI 能用是因为做了兼容，但实际上这部分不属于UEFI的定义了）；
    4) 输出也不再是单纯的二进制code，改为Removable Binary Drivers；
    5) OS启动不再是调用Int19，而是直接利用protocol/device Path；
    6) 对于第三方的开发，前者基本上做不到，除非参与BIOS的设计，但是还要受到ROM的大小限制，而后者就便利多了。
    7) 弥补BIOS对新硬件的支持不足的问题。
 2. 描述PXE的大致启动流程。
 
   1) 客户端个人电脑开机后， 在 TCP/IP Bootrom 获得控制权之前先做自我测试。
    2)Bootprom 送出 BOOTP/DHCP 要求以取得 IP。
    3) 如果服务器收到个人电脑所送出的要求， 就会送回 BOOTP/DHCP 回应，内容包括
   客户端的 IP 地址， 预设网关， 及开机映像文件。否则，服务器会忽略这个要求。
    4)Bootprom 由 TFTP 通讯协议从服务器下载开机映像文件。
    5) 个人电脑通过这个开机映像文件开机， 这个开机文件可以只是单纯的开机程式也可
    以是操作系统。
    6) 开机映像文件将包含 kernel loader 及压缩过的 kernel，此 kernel 将支持NTFS root
   系统。
    7) 远程客户端根据下载的文件启动机器。

## 3.2 系统启动流程
 1. 了解NTLDR的启动流程。

    NTLDR文件的是一个隐藏的，只读的系统文件，位置在系统盘的根目录，用来装载操作系统。
    一般情况系统的引导过程是这样的代码
    1：电源自检程序开始运行
    2：主引导记录被装入内存，并且程序开始执行
    3：活动分区的引导扇区被装入内存
    4：NTLDR从引导扇区被装入并初始化
    5：将处理器的实模式改为32位平滑内存模式
    6：NTLDR开始运行适当的小文件系统驱动程序。
    小文件系统驱动程序是建立在NTLDR内部的，它能读FAT或NTFS。
    7：NTLDR读boot.ini文件
    8：NTLDR装载所选操作系统
    如果windows NT/windows 2000/windows XP/windows server 2003这些操作系统被选择，NTLDR运行Ntdetect。
    对于其他的操作系统，NTLDR装载并运行Bootsect.dos然后向它传递控制。
    windows NT过程结束。
    9：Ntdetect搜索计算机硬件并将列表传送给NTLDR，以便将这些信息写进\\HKE Y_LOCAL_MACHINE\HARDWARE中。
    10：然后NTLDR装载Ntoskrnl.exe，Hal.dll和系统信息集合。
    11：Ntldr搜索系统信息集合，并装载设备驱动配置以便设备在启动时开始工作
    12：Ntldr把控制权交给Ntoskrnl.exe，这时，启动程序结束，装载阶段开始
 2. 了解GRUB的启动流程。

    当系统加电后，固化在BIOS中的程序首先对系统硬件进行自检，自检通过后，就加载启动磁盘上的MBR，并将控制权交给MBR中的程序(stage1)，stage1执行，判断自己是否GRUB，如果是且配置了stage1_5，则加载stage1_5，否则就转去加载启动扇区，接着，stage2被加载并执行，由stage2借助stage1_5驱动文件系统，并查找grub.conf，显示启动菜单供用户选择，然后根据用户的选择或默认配置加载操作系统内核，并将控制权交给操作系统内核，由内核完成操作系统的启动。
    GRUB涉及到几个重要的文件：
    第一个就是stage1。它被安装在MBR扇区（0面0磁道的第1扇区），大小为512字节（446字节代码+64字节分区表+2字节标志55AA），它负责加载存放于0面0道第2扇区的start程序。
    第二个是stage1_5。stage1_5负责识别文件系统和加载stage2，所以stage1_5往往有多个，以支持不同文件系统的读取。在安装GRUB的时候，GRUB会根据当前/boot/分区类型，加载相应的stage1_5到0面0磁道的第3扇区。stage1_5是由start加载的。
    第三个是stage2。它负责显示启动菜单和提供用户交互接口，并根据用户选择或默认配置加载操作系统内核。同前两个文件不同，stage2是存放在磁盘上/boot/grub下。
    第四个是menu.lst(/boot/grub/grub.conf的链接)。grub.conf是一个基于脚本的文本文件，其中包含菜单显示的配置和各个操作系统的内核加载配置。GRUB根据grub.conf显示启动菜单，提供同用户交互界面。GRUB正是根据用户选择或默认配置和grub.conf的内核配置加载相应的内核程序，并把控制权交给内核程序，使得内核程序完成真正的操作系统的启动。
    其它重要文件，GRUB除了上面叙述的主要文件之外，还包括支持交互功能的一些磁盘程序。主要包括/sbin/下的grub、grub-install、grub-md5-crypt和grub-terminfo和/usr/bin/mbchk，以及/boot/grub下的设备映像文件(device.map)和菜单背景图像文件(splash.xpm.gz)。
 3. 比较NTLDR和GRUB的功能有差异。

    ntldr功能很少，只能引导win，只能装在硬盘；
    grub是第三方操作系统引导器，
    可以引导硬盘，光盘，网络，U盘，
    winxp，winpe，win7，linux，dos，……
 4. 了解u-boot的功能。

   u-boot是常用的嵌入式操作系统启动程序。著名的开源bootloader程序。可以启动linux、android等系统。
   作为bootloader它的最基本的作用为：
   1、把操作系统镜像从介质如flash、nand、SD卡等加载到内存
   2、在内存中把操作系统启动，启动时可以向操作系统传递启动配置信息。
   当然它还有一个简单的控制台，利用串口与用户交互以提供一些额外的辅助功能，如在OS启动前查看内存、数据拷贝、查看OS镜像信息、检查坏块等

## 3.3 中断、异常和系统调用比较
1. 什么是中断、异常和系统调用？
2. 中断、异常和系统调用的处理流程有什么异同？
3. 举例说明Linux中有哪些中断，哪些异常？

    中断：敲键盘
    异常：除0，访问无效地址
4. 以ucore lab8的answer为例，uCore的系统调用有哪些？大致的功能分类有哪些？(w2l1) 

  __alltraps:
    # push registers to build a trap frame
    # therefore make the stack look like a struct trapframe
    pushl %ds
    pushl %es
    pushl %fs
    pushl %gs
    pushal
    # load GD_KDATA into %ds and %es to set up data segments for kernel
    movl $GD_KDATA, %eax
    movw %ax, %ds
    movw %ax, %es
    # push %esp to pass a pointer to the trapframe as an argument to trap()
    pushl %esp
    接着调用trap()函数。
    call trap
    trap()函数中调用trap_dispatch(),根据不同类型的中断来处理
    时钟中断的处理如下：
    ticks ++;
    assert(current != NULL);
    run_timer_list();
5. Linux的系统调用有哪些？大致的功能分类有哪些？  (w2l1)

    linux的系统调用大约有250个。系统调用的功能分类如下：
    - 进程控制
    - 文件系统控制
      。文件读写操作
      。文件系统操作
    - 系统控制
    - 内存管理
    - 网络管理
    - socket控制
    - 用户管理
```
  + 采分点：说明了Linux的大致数量（上百个），说明了Linux系统调用的主要分类（文件操作，进程管理，内存管理等）
  - 答案没有涉及上述两个要点；（0分）
  - 答案对上述两个要点中的某一个要点进行了正确阐述（1分）
  - 答案对上述两个要点进行了正确阐述（2分）
  - 答案除了对上述两个要点都进行了正确阐述外，还进行了扩展和更丰富的说明（3分）
 ```
 
6. 以ucore lab8的answer为例，uCore的系统调用有哪些？大致的功能分类有哪些？(w2l1)

ucore的系统调用有22个。 可见syscall/syscall.c
具体为：
SYS_exit  SYS_fork SYS_wait SYS_exec SYS_yield SYS_kill SYS_getpid SYS_putc SYS_pgdir SYS_gettime SYS_lab6_set_priority SYS_sleep SYS_open SYS_close SYS_read
SYS_write SYS_seek SYS_fstat SYS_fsync SYS_getcwd SYS_getdirentry SYS_dup
ucore的功能分类主要有：
进程控制 exit fork wait exec yield kill getpid sleep lab6_set_priority 
文件系统控制 pgdir open close read wirte seek fstat fsync getcwd getdirentry dup
系统控制 gettime putc

## 3.4 linux系统调用分析
 1. 通过分析[lab1_ex0](https://github.com/chyyuu/ucore_lab/blob/master/related_info/lab1/lab1-ex0.md)了解Linux应用的系统调用编写和含义。(w2l1)
 

 ```
  + 采分点：说明了objdump，nm，file的大致用途，说明了系统调用的具体含义
  - 答案没有涉及上述两个要点；（0分）
  - 答案对上述两个要点中的某一个要点进行了正确阐述（1分）
  - 答案对上述两个要点进行了正确阐述（2分）
  - 答案除了对上述两个要点都进行了正确阐述外，还进行了扩展和更丰富的说明（3分）
 
 ```
 
 2. 通过调试[lab1_ex1](https://github.com/chyyuu/ucore_lab/blob/master/related_info/lab1/lab1-ex1.md)了解Linux应用的系统调用执行过程。(w2l1)
 

 ```
  + 采分点：说明了strace的大致用途，说明了系统调用的具体执行过程（包括应用，CPU硬件，操作系统的执行过程）
  - 答案没有涉及上述两个要点；（0分）
  - 答案对上述两个要点中的某一个要点进行了正确阐述（1分）
  - 答案对上述两个要点进行了正确阐述（2分）
  - 答案除了对上述两个要点都进行了正确阐述外，还进行了扩展和更丰富的说明（3分）
 ```
 
## 3.5 ucore系统调用分析
 1. ucore的系统调用中参数传递代码分析。
 2. 以getpid为例，分析ucore的系统调用中返回结果的传递代码。
 3. 以ucore lab8的answer为例，分析ucore 应用的系统调用编写和含义。
 4. 以ucore lab8的answer为例，尝试修改并运行ucore OS kernel代码，使其具有类似Linux应用工具`strace`的功能，即能够显示出应用程序发出的系统调用，从而可以分析ucore应用的系统调用执行过程。
 
## 3.6 请分析函数调用和系统调用的区别
 1. 请从代码编写和执行过程来说明。
   1. 说明`int`、`iret`、`call`和`ret`的指令准确功能
 

## v9-cpu相关题目
---

### 提前准备
```
cd YOUR v9-cpu DIR
git pull 
cd YOUR os_course_spoc_exercise DIR
git pull 
```

### v9-cpu系统调用实现
  1. v9-cpu中os4.c的系统调用中参数传递代码分析。
  1. v9-cpu中os4.c的系统调用中返回结果的传递代码分析。
  1. 理解v9-cpu中os4.c的系统调用编写和含义。

