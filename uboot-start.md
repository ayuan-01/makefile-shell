# 使用正点原子官方工具烧写

恩智浦提供了官方的OTG模式的镜像文件烧写软件**mfgtool**。

下载官方mfgtool工具，里面有两个压缩包，后缀分别是`without-rootfs`和`with-rootfs`。我们选择带根文件系统的。对其解压会出现一系列文件。

<img src="./fig/wechat_2025-08-06_175133_452.png" alt="tree mfgtools">

在上面的文件目录中，重点关注**Profiles**这个文件夹`./Profile/linux/OS Firmware`，我们的烧写文件就放到这个文件夹中。`Mfgtool2.exe`就是烧写软件。要烧写到什么芯片呢，烧写什么介质中，这些通过`vbs`文件来配置。正点原子开发板使用的是emmc的存储介质，使用的是`mfgtool2-yocto-mx-evk-emmc.vbs`。

烧写的时候连接开发板的`USB-OTG1`接口，启动方式设置为`USB`模式，拔出SD卡。按下开发板复位键PC就会识别到块设备。

开发板与电脑连接之后打开`vbs`，出现下面的页面,点击start即可开始烧写。

<img src="./fig/image-20250806222109375.png">

我们烧写的文件有四个：**uboot, zImage, dtb, rootfs**。现在要确定这四个文件要放到哪里。`./Profile/linux/OS Firmware`中有两个文件夹`files` `firmware`以及文件`ucl2.xml`，mfgtool烧写分为两个阶段。

1. `firmware`文件夹存在`uboot zImage dtb`通过USB OTG将这个文件下载到开发板的`DDR`中，在`DDR`中启动Linux系统，为后面的烧写做准备。
2. `files`文件夹中存放着最终要烧录的`uboot zImage dtb rootfs`文件，经过第一步的操作，开发板上已经运行着一个Linux系统了，此时可以完成对EMMC的格式化，分区操作，EMMC分区建立好之后就可以从`files`文件夹中读取要烧写的文件，烧写到EMMC中。
3. `ucl2.xml`文件配置向什么介质烧写系统，向什么芯片烧写系统。

上述两步中都是将自己编译出来的文件改名，然后对目标文件进行一个替换。

<img src="./fig/image-20250806224952806.png">

# 使用SD卡烧写

```sh
./imxdownload uboot.bin /dev/sdb

# uboot启动后配置环境变量
=> fatls mmc  1:1
  6785480   zimage 
    39459   imx6ull-14x14-emmc-4.3-480x272-c.dtb 
    39459   imx6ull-14x14-emmc-4.3-800x480-c.dtb 
    39459   imx6ull-14x14-emmc-7-800x480-c.dtb 
    39459   imx6ull-14x14-emmc-7-1024x600-c.dtb 
    39459   imx6ull-14x14-emmc-10.1-1280x800-c.dtb 

8 file(s), 0 dir(s)
# 前者是设置镜像和设备树从什么位置启动，后者是设置终端和根文件系统
bootcmd='fatload mmc 1:1 80800000 zImage;fatload mmc 1:1 830000000 imx6ull-14x14-emmc-7-1024x600-c.dtb; bootz 80800000 - 83000000'
bootargs='console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw'

mmcblk1p2:
    1：设备的编号（从 0 开始计数）。
    mmcblk0：通常是系统内置的 eMMC 或第一个 SD 卡插槽。
    mmcblk1：第二个 MMC 设备（如外接 SD 卡或第二个 eMMC）。
    p2：分区的编号（从 1 开始计数）。
    p1：第一个分区（如 /boot 或 EFI 系统分区）。
    p2：第二个分区（通常是根文件系统 / 或用户数据分区）。
```

```sh
# 为SD卡创建分区
sudo fdisk -l
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0 238.5G  0 disk 
└─sda1        8:1    0 238.5G  0 part /
mmcblk0     179:0    0  29.8G  0 disk 
├─mmcblk0p1 179:1    0   256M  0 part /boot
└─mmcblk0p2 179:2    0  29.5G  0 part /home
mmcblk1     179:8    0  14.9G  0 disk  # 这是新插入的 SD 卡（未分区）

sudo fdisk /dev/mmcblk1  # 替换为你的 SD 卡设备路径

# 创建第一个分区（如 512MB 的 FAT32 引导分区）
Command (m for help): n		# 输入 n 创建新分区
Partition type: p			# 主分区（primary）
Partition number: 1			# 分区号（默认 1）
First sector: 2048			# 起始扇区（默认 2048，直接回车）
Last sector: +512M  		# 结束扇区512MB

# 2048是逻辑扇区号，指的是从磁盘开头数第 2048 个逻辑扇区。
# 物理扇区和逻辑扇区的区别，目前物理扇区大小为 4096 字节（即 4KB）。为了保持与旧操作系统的兼容性，操作系统会模拟512字节的逻辑扇区。
 
# 创建第二个分区（剩余空间）
Command (m for help): n
Partition type: p
Partition number: 2
First sector: (直接回车，使用默认值)
Last sector: (直接回车，使用剩余空间)

Command (m for help): t  # 修改分区类型
Partition number: 1     # 选择分区
Hex code (type L to list): 1  # 输入类型代码（1=FAT32，83=Linux，EF=EFI）

Command (m for help): w  # 写入并退出

sudo mkfs.vfat -F 32 /dev/mmcblk1p2  # 格式化第二个分区为 FAT32

sudo mkdir /mnt/sdcard  # 创建挂载点
sudo mount /dev/mmcblk1p1 /mnt/sdcard  # 挂载分区
```

# 使用tftp网络文件系统烧写

```sh
# 使用tftp烧写系统，使用nfs挂载根文件系统
bootcmd="tftp 80800000 zImage;tftp 83000000 imx6ull-alientek-emmc.dtb;bootz 80800000 - 83000000"
bootargs="console=ttymxc0,115200 root=/dev/nfs rw nfsroot=192.168.1.250:/home/zuozhongkai/linux/nfs/rootfs ip=192.168.1.251:192.168.1.250:192.168.1.1:255.255.255.0::eth0:off"

# /lib/modules/4.1.15
```

# 裸机的启动

boot ROM会对**系统时钟**进行初始化，`System PLL=528Mhz，USB PLL=480MHz，AHB=132MHz，IPG=66MHz。`在下载镜像的时候`L1 ICache`会打开，验证镜像的时候`L1 DCache、L2 Cache`和`MMU`都会打开。验证完成之后全部关闭。中断向量偏移会设置到boot ROM的起始位置，用户代码被启动后中断向量偏移会被重新设置，一般是设置到用户代码开始的时候。

在设置好启动方式时，如SD卡启动，root ROM会到SD卡的指定位置寻找裸机程序，因此烧录程序`imxdownload `会把裸机程序烧录到SD卡的指定位置，如下图。IVT的偏移为1KB，IVT+Boot data+DCD的总长为4KB-1KB=3KB。

<img src="./fig/image-20250807164335757.png">

以uboot为例，因为uboot本身是一个庞大的裸机项目。i.MX6ULL的裸机编程涉及三个文件：`start.S` `uboor.bin` `lds` `imx`。

向SD卡烧写.bin文件的命令：`./imxdownload led.bin /dev/sdd` 。这个命令会生成一个`load.imx`文件，相比于.bin文件多出了一些头部信息。那么这个头部信息是什么呢？

## IMX文件

`.imx`文件的组成如下：

1. IVT(Image vector table)。包含一系列地址信息，在ROM按照固定的地址存放着。
   1. entry定义了程序的入口地址0X87800000，从链接脚本文件定义而来。
   2. dcd保存了DCD地址。
   3. boot data保存了boot data的地址
2. Boot data。包含镜像要拷贝到DDR的哪个地址，拷贝的大小是多少。
   1. start保存着整个load.imx的起始地址0X877FF000。
3. DCD(Device configuration data)。DDR的初始化配置信息。
4. bin文件。实际的可执行文件。

## LD文件

首先，链接脚本指定了cpu从DDR中读取指令的位置。或者说把可执行文件从ROM放到DDR中的位置。分配了各个区的地址。

```sh
SECTIONS{
    . = 0X87800000;							# 链接起始地址为0X87800000
    .text :									# 代码段地址0X87800000，存放顺序为start.o main.o 
    {
        start.o 
        main.o 
        *(.text)
    }
    .rodata ALIGN(4) : {*(.rodata*)} 		# 常量区
    .data ALIGN(4) : { *(.data) } 			# 初始化的全局变量
    __bss_start = .; 						# 静态变量区
    .bss ALIGN(4) : { *(.bss) *(COMMON) } 
    __bss_end = .;
}
```

## start.S

建立中断向量表

初始化CPU工作模式

分配栈指针

跳转main函数

提供基本的 IRQ 中断处理框架

## 裸机程序编译烧写的命令

```shell
arm-linux-gnueabihf-ld -Timx6ul.lds -o ledc.elf start.o main.o
arm-linux-gnueabihf-objcopy -O binary -S ledc.elf ledc.bin
arm-linux-gnueabihf-objdump -D -m arm ledc.elf > ledc.dis

# 第一个命令是把start.o和main.o链接为elf文件
# 第二个命令把elf文件objcopy为bin文件
# 第三个命令反编译为汇编
```

# bin文件和elf文件的区别

首先elf可执行文件通常运行在操作系统之上。bin文件通常用于嵌入式裸机程序的烧录。

bin文件是存粹的机器码，没有说明书。bin文件是赤裸的，它假设加载它的环境（如嵌入式系统的引导程序）已经预先知道了**代码存放的地址，代码的入口，数据段，代码段的地址。**

elf文件是带有详细说明书和装配图的文件。elf文件有以下几个部分**ELF头部，程序头表，节区，节头表**

1. ELF头部

   1. 标识这是一个ELF文件，文件类型可重定位文件还是共享库文件，ARM架构。

   2. e_entry：程序的入口地址，第一条指令的虚拟地址。

      **对于动态链接的程序，这个地址通常不是 `main` 函数，而是动态链接器 `_start` 的入口**。

   3. 程序头表和节头表的偏移。

2. 程序头表

   加载说明书，用于告诉操作系统如何将elf文件的内容的映射到内存中，以创建一个进程。

   程序头表是一个结构体数组，每个结构体（segment）描述了文件中的一块区域应该如何被映射到内存中。

   1. **PT_LOAD**：代码段，数据段等需要被加载到内存中的段。
   2. **PT_DYNAMIC**：包含动态链接信息的段。
   3. **PT_INTERP**：动态链接器的路径（lib/ld-linux.so.2）。

   每个程序头都包含了该段在文件中的偏移，在内存中的虚拟地址，大小，执行权限等。

3. 节区

   链接器和调试工具的零件清单，保存着不同节的具体内容。

   1. `.text`：存放程序指令（代码）。
   2. `.data`：存放已初始化的全局变量和静态变量。
   3. `.bss`：存放未初始化的全局变量和静态变量（此节在文件中不占空间，只在运行时在内存中分配）。
   4. `.rodata`：存放只读数据（如常量字符串）。
   5. `.symtab`：符号表，记录了函数和变量的名称及其地址。
   6. `.strtab`：字符串表，存放符号名、文件名等字符串。
   7. `.rel.text` / `.rel.data`：重定位信息，用于链接时修正地址。

4. 节头表

   所有节的索引目录。存放一个数组，数组中的每个元素对应一个节，描述了该节的名称（在`.strtab`中的索引），类型，在文件中的偏移，大小，链接信息等。

### elf文件的加载过程

1. shell调用，shell会fork一个子进程，然后execve跳转到elf可执行文件中。
2. 跳转到内核
   1. 读取**elf头部**。验证是不是elf文件，读取头部信息，找到程序头表。
   2. 解析**程序头表**。寻找PT_LOAD段，这是唯一需要被实际加载到内存中的段，通常一个ELF文件至少有两个PT_LOAD段，即代码段和数据段。
   3. 在找到**PT_LOAD**段之后，加载器会为当前进程创建一个新的虚拟内存区域，起始地址和大小`PT_LOAD->p_vaddr`和`PT_LOAD->p_memsz`决定。设置权限。
   4. 上面的步骤建立了进程的虚拟内存区域，现在需要完成**虚拟内存区域到物理内存的映射**。需要注意的是这个映射并不是把磁盘的所有内容都直接复制到内存里面，而是在MMU触发缺页中断的时候才从磁盘中把需要的数据放入内存。
   5. 处理**.bss段**，其在二进制文件中不占用空间，但是需要在内存中为其分配`p_memsz`的空间。
   6. 寻找程序头表中的**PT_INTERP**段，其存放着**动态链接器**的路径，加载器会把这个动态链接器也放到进程的虚拟内存中。
   7. 设置栈。内核会在进程空间地址顶端创建一个栈区域，压入一些数据，传入参数个数，传入参数指针，指向各环境变量的字符串等和一些辅助向量。
      - `AT_ENTRY`：程序的入口点地址。
      - `AT_PHDR`：程序头表在内存中的地址。
      - `AT_PHENT`：程序头表中每个条目的大小。
      - `AT_PHNUM`：程序头表中条目的数量。
      - `AT_RANDOM`：16字节的随机数，用于地址空间布局随机化，增加安全性。
      - ...
3. 从内核跳转到用户空间
   1. 内核将指令指针PC设置为**动态链接器**的入口。栈指针指向刚刚的栈顶，切换到用户模式。
   2. **动态链接器 `_start` 函数开始运行**。
   3. 读取主程序的 `.dynamic` 节区，找到程序依赖的共享库列表，加载这些库。
   4. 上一步动态库加载了之后就有了地址，此时就需要对库函数地址（占位符）进行替换为真实的虚拟地址。
   5. 执行主程序和各个共享库的初始化代码。
   6. 跳转main函数执行。

## 符号表，字符串表和重定位信息

符号表，字符串表，重定位信息都属于节区。

#### 静态链接

静态链接情况下，在可执行文件编译完成的时候，重定位已经完成，节区中重定位信息被删除或者为空，符**号表通常会保留，用于GDB调试。**

符号表`.symtab`与字符串`strtab`表结合作用，符号表中保存着程序中**变量，函数**，（文件名，节区名）等的**名称**（索引`st_name`，指向`strtab`中的对应位置），**地址**（`st_value`，通常是函数，变量的地址或者偏移量），大小（st_size，一个数组或者函数的字节大小），类型（st_info，如 `STT_FUNC` 函数, `STT_OBJECT` 对象，绑定属性：`STB_LOCAL`：局部符号；`STB_GLOBAL`：全局符号；`STB_WEAK`：弱符号；`st_shndx`：一个索引，指明该符号位于哪个节区。）

为什么不直接把字符串存在 `.symtab` 里？因为这样做效率低下且浪费空间。使用索引的方式，多个符号可以共享同一个字符串（例如，多个文件都引用 `printf`），并且符号表条目可以保持固定大小，便于快速查找。

重定位表（`rel.text/rel.data`）要解决的问题：当存在多个.c文件时，一个c文件使用到另一个c文件的函数，在编译单个c文件时，编译器并不知道调用的这个函数的最终地址，也不知道自己定义的函数或者变量最终会被链接到可执行文件的哪个地址。这时**汇编器**会生成一个重定位条目，并留下一个占位符，表示这个位置的代码需要被修正。重定位条目生成在汇编阶段，最终地址的确定发生在链接阶段。每个重定位条目（通常是一个结构体变量，Elf_Rel或Elf_Rela）包含**需要被修正的地址在节点中的偏移量r_offset**，**r_info，存放符号索引和重定位类型**。*对某个函数和变量进行重定位，首先是要知道这个需要被重定位的函数变量在哪（r_offset），其次是要知道要填充进这个占位符的地址是什么（从符号表中获得r_info）*。

两种重定位表的介绍：

- **`.rel.text`**：包含了对**代码节区 (`.text`)** 的重定位信息。例如，`call printf` 指令中的 `printf` 函数地址，在编译时是未知的，这里就会生成一个重定位条目。
- **`.rel.data`**：包含了对**已初始化数据节区 (`.data`)** 的重定位信息。例如，`int *ptr = &global_var;` 这行代码，`ptr` 变量在 `.data` 节区，但它存储的 `global_var` 的地址在编译时也是未知的，这里也会生成一个重定位条目。

对于一个工程文件的编译流程，预处理，编译，汇编，链接。假设一个简单的工程文件包括a.c，b.c包括静态库文件.a。在汇编阶段，汇编器会对各个.c文件进行汇编，由于这时各个文件中的函数变量在可执行文件中的地址并没有被确定，会生成很多重定位条目。**在链接阶段**，链接器ld会读取所有的.o文件和静态库文件.a，把所有同类型的**节区合并**，读取各个.o文件的符号表，创建一个全局符号表，并且在这个过程中进行符号解析。此时整个可执行文件的地址，符号表基本确定，需要根据重定位条目对一些占位符进行重定位处理。

1. 根据r_info找到符号表中的对应位置，获取该符号的最终地址。
2. 分局r_info中的重定位类型，计算出需要写入的值。
3. 找到r_offset指定的需要重定位的占位符的位置，执行重定位。

链接完成之后，`.text` 和 `.data` 中的地址引用都是完整的、可以直接运行的。**`.rel.text` 和 `.rel.data` 节区通常会被丢弃**，因为所有重定位工作已经完成，不再需要它们了。`.symtab` 和 `.strtab` 可能会被保留（用于调试），或者被 `strip` 工具移除以减小文件大小。

#### 动态链接

首先，根据elf加载流程，动态链接的地址重定位是在可执行文件执行过程中在内核分配完内存空间和栈空间之后，调用ld.so动态连接器，转到用户空间加载依赖的共享库，然后进行运行时地址重定位。最终跳转程序入口，开始执行程序。

链接过程：

1. 处理内部符号的重定位。
2. 对于外部符号，因为地址并不确定，所以链接器并不会解析它的最终地址。
3. 链接器为外部符号生成动态重定位信息，保存在.rela.plt（函数）和.rela.dyn（数据）节区中，对应的动态符号表.dynsym是.symtab的一个子集，只包含用于动态链接的全局符号。动态字符串表为.dynstr。

程序加载过程：

1. 调用动态链接器ld.so，并执行。
2. 根据程序头表中的PT_DYNAMIC找到.dynamic节区，遍历所有的DT)NEEDED条目，加载所有依赖的共享库文件，此时主程序所用的库文件都有了自己的地址。然后根据DT_RELA`、`DT_RELASZ`、`DT_JMPREL告诉动态链接器.rela.dyn和.rela.plt的位置。
3. 处理数据重定位.rela.dyn。
4. 处理函数重定位.rela.plt。使用延迟绑定策略，当函数第一次被调用时，控制权会转到PLT中，PLT代码会触发ld.so解析真正的函数地址。解析完成后写入PLT，后续调用查表即可。

## 流程图

<img src="./fig/exe_flow.png" width="800">

# 裸机中断例程

## 汇编部分

下面的汇编文件做的事情：

1. 创建中断向量表，`ldr pc, =Reset_Handler`是一个伪指令，相当于把Reset_Handler这个中断处理程序放到pc中，也就是跳转到这个Reset_Handler处去执行。创建中断向量表的代码在程序入口处，每一条都对应着进入某一个中断处理程序的起始地址，当某个中断发生的时候，跳转到中断向量表的对应的那个伪指令执行即可。

   | 地址   | 指令                       | 含义           |
   | :----- | :------------------------- | :------------- |
   | `0x00` | `ldr pc, ResetHandler`     | 复位中断       |
   | `0x04` | `ldr pc, UndefinedHandler` | 未定义指令中断 |
   | `0x08` | `ldr pc, SVCHandler`       | SVC 中断       |
   | `0x0C` | `ldr pc, PrefAbortHandler` | 预取终止中断   |
   | `0x10` | `ldr pc, DataAbortHandler` | 数据终止中断   |
   | `0x14` | `ldr pc, NotUsedHandler`   | 未使用中断     |
   | `0x18` | `ldr pc, IRQHandler`       | IRQ 中断       |
   | `0x1C` | `ldr pc, FIQHandler`       | FIQ 中断       |

2. 复位中断处理程序。

   1. 操作协处理器p15的c1，**系统控制寄存器（SCTLR，System Control Register）**，关闭 I-Cache、关闭 D-Cache、关闭数据对齐检查，关闭分支预测，关闭 MMU，写回 SCTLR 寄存器。

      `mrc`：**Move to Register from Coprocessor**，表示从协处理器读取数据到通用寄存器。

      `p15`：**协处理器编号 15**，ARM 架构中，CP15 负责系统控制（如 MMU、缓存、权限控制等）。

      `0`：**操作码 1 (Opcode1)**，通常为 0，用于区分不同的协处理器操作。

      `<Rd>`：**目标寄存器**，读取的值会存储到这个通用寄存器（如 `r0`）。

      `c1`：**协处理器主寄存器 (CRn)**，这里 `c1`表示 **系统控制寄存器 (SCTLR)**。

      `c0`：**协处理器辅助寄存器 (CRm)**，通常为 `c0`，用于进一步细分功能。

      `0`：**操作码 2 (Opcode2)**，通常为 0，用于区分不同的子操作。 

   2. 设置向量表基地址，在 ARMv7 及以上架构中，向量表基地址由 VBAR 寄存器控制。ldr r0, =0X87800000，mcr p15, 0, r0, c12, c0, 0。
   3. 初始化CPU各个模式栈空间，设置sp地址。
   4. cpsie i打开全局中断。允许IRQ中断。
   5. 跳转到main函数

3. 定义不同中断的处理入口

   SVC_Handler: ldr r0, =SVC_Handler bx r0; 这里跳转的是C语言中定义的中断处理函数。

   b表示直接跳转，bx表示跳转并切换指令集，blx表示跳转，返回（bx lr）并切换指令集。

4. IRQ中断处理

   - **保存 CPU 现场寄存器**（lr、r0-r3、r12、spsr）
   - **通过 GIC（通用中断控制器）读取中断号**
   - **切换到 SVC 模式，允许嵌套中断**
   - **调用 C 语言中断处理函数**
   - **恢复现场并返回中断前的程序流**

   总体来说就是保护现场，使用GIC读取中断号，切换SVC允许中断嵌套，调用C语言中断处理函数，返回现场。

```assembly
.global _start /* 全局标号 */

/*
* 描述： _start 函数，首先是中断向量表的创建
*/
_start:
	ldr pc, =Reset_Handler 		/* 复位中断 */ 
	ldr pc, =Undefined_Handler 	/* 未定义指令中断 */
	ldr pc, =SVC_Handler 		/* SVC(Supervisor)中断*/
	ldr pc, =PrefAbort_Handler 	/* 预取终止中断 */
	ldr pc, =DataAbort_Handler 	/* 数据终止中断 */
	ldr pc, =NotUsed_Handler 	/* 未使用中断 */
	ldr pc, =IRQ_Handler 		/* IRQ 中断 */
	ldr pc, =FIQ_Handler 		/* FIQ(快速中断) */

/* 复位中断 */ 
Reset_Handler:

	cpsid i 					/* 关闭全局中断 */

	/* 关闭 I,DCache 和 MMU 
	* 采取读-改-写的方式。
	*/
	mrc p15, 0, r0, c1, c0, 0	/* 读取 CP15 的 C1 寄存器到 R0 中 */
	bic r0, r0, #(0x1 << 12) 	/* 清除 C1 的 I 位，关闭 I Cache */
	bic r0, r0, #(0x1 << 2) 	/* 清除 C1 的 C 位，关闭 D Cache */
	bic r0, r0, #0x2 			/* 清除 C1 的 A 位，关闭对齐检查 */
	bic r0, r0, #(0x1 << 11) 	/* 清除 C1 的 Z 位，关闭分支预测 */
	bic r0, r0, #0x1 			/* 清除 C1 的 M 位，关闭 MMU */
	mcr p15, 0, r0, c1, c0, 0 	/* 将 r0 的值写入到 CP15 的 C1 中 */


#if 0
	/* 汇编版本设置中断向量表偏移 */
	ldr r0, =0X87800000

	dsb
	isb
	mcr p15, 0, r0, c12, c0, 0
	dsb
	isb
#endif

	/** 
	 * 设置各个模式下的栈指针，
	 * 注意：IMX6UL 的堆栈是向下增长的！
	 * 堆栈指针地址一定要是 4 字节地址对齐的！！！
	 * DDR 范围:0X80000000~0X9FFFFFFF 或者 0X8FFFFFFF
	 */
	/* 进入 IRQ 模式 */
	mrs r0, cpsr
	bic r0, r0, #0x1f 		/* 将 r0 的低 5 位清零，也就是 cpsr 的 M0~M4 */
	orr r0, r0, #0x12 		/* r0 或上 0x12,表示使用 IRQ 模式 */
	msr cpsr, r0 			/* 将 r0 的数据写入到 cpsr 中 */
	ldr sp, =0x80600000 	/* IRQ 模式栈首地址为 0X80600000,大小为 2MB */

	/* 进入 SYS 模式 */
	mrs r0, cpsr
	bic r0, r0, #0x1f 		/* 将 r0 的低 5 位清零，也就是 cpsr 的 M0~M4 */
	orr r0, r0, #0x1f 		/* r0 或上 0x1f,表示使用 SYS 模式 */
	msr cpsr, r0 			/* 将 r0 的数据写入到 cpsr 中 */
	ldr sp, =0x80400000 	/* SYS 模式栈首地址为 0X80400000,大小为 2MB */

	/* 进入 SVC 模式 */
	mrs r0, cpsr
	bic r0, r0, #0x1f 		/* 将 r0 的低 5 位清零，也就是 cpsr 的 M0~M4 */
	orr r0, r0, #0x13 		/* r0 或上 0x13,表示使用 SVC 模式 */
	msr cpsr, r0 			/* 将 r0 的数据写入到 cpsr 中 */
	ldr sp, =0X80200000 	/* SVC 模式栈首地址为 0X80200000,大小为 2MB */

	cpsie i 				/* 打开全局中断 */

#if 0
	/* 使能 IRQ 中断 */
	mrs r0, cpsr 			/* 读取 cpsr 寄存器值到 r0 中 */
	bic r0, r0, #0x80 		/* 将 r0 寄存器中 bit7 清零，也就是 CPSR 中
							 * 的 I 位清零，表示允许 IRQ 中断
							 */
	msr cpsr, r0 /* 将 r0 重新写入到 cpsr 中 */
#endif

b main /* 跳转到 main 函数 */

/*
ldr r0, =Undefined_Handler 加载的是 Undefined_Handler 的地址，而不是 Undefined_Handler 标号本身。
bx r0 跳转到的是 Undefined_Handler 的地址，也就是 跳转到 C 语言或其他地方定义的中断处理函数。
换句话说，这段代码的作用是：跳转到 C 语言实现的中断处理函数。
b指令只能跳转到 当前文件内的标号，无法跳转到 C 函数（因为 C 函数地址由链接器决定）。
*/

/* 未定义中断 */
Undefined_Handler:
	ldr r0, =Undefined_Handler
	bx r0

/* SVC 中断 */
SVC_Handler:
	ldr r0, =SVC_Handler
	bx r0

/* 预取终止中断 */
PrefAbort_Handler:
	ldr r0, =PrefAbort_Handler 
	bx r0

/* 数据终止中断 */
DataAbort_Handler:
	ldr r0, =DataAbort_Handler
	bx r0

/* 未使用的中断 */
NotUsed_Handler:

	ldr r0, =NotUsed_Handler
	bx r0

/* IRQ 中断！重点！！！！！ */
IRQ_Handler:
	push {lr} 			/* 保存 lr 地址 */
	push {r0-r3, r12} 	/* 保存 r0-r3，r12 寄存器 */

	mrs r0, spsr 		/* 读取 spsr 寄存器 */
	push {r0} 			/* 保存 spsr 寄存器 */

	mrc p15, 4, r1, c15, c0, 0 	/* 将 CP15 的 C0 内的值到 R1 寄存器中
	* 参考文档 ARM Cortex-A(armV7)编程手册 V4.0.pdf P49
	* Cortex-A7 Technical ReferenceManua.pdf P68 P138
	*/ 
	add r1, r1, #0X2000 		/* GIC 基地址加 0X2000，得到 CPU 接口端基地址 */
	ldr r0, [r1, #0XC] 			/* CPU 接口端基地址加 0X0C 就是 GICC_IAR 寄存器，
								* GICC_IAR 保存着当前发生中断的中断号，我们要根据
								* 这个中断号来绝对调用哪个中断服务函数
								*/
	push {r0, r1} 				/* 保存 r0,r1 */

	cps #0x13 					/* 进入 SVC 模式，允许其他中断再次进去 */

	push {lr} 					/* 保存 SVC 模式的 lr 寄存器 */
	ldr r2, =system_irqhandler 	/* 加载 C 语言中断处理函数到 r2 寄存器中*/
	blx r2 						/* 运行 C 语言中断处理函数，带有一个参数 */

	pop {lr} 					/* 执行完 C 语言中断服务函数，lr 出栈 */
	cps #0x12 					/* 进入 IRQ 模式 */
	pop {r0, r1} 
	str r0, [r1, #0X10] 		/* 中断执行完成，写 EOIR */

	pop {r0} 
	msr spsr_cxsf, r0 			/* 恢复 spsr */

	pop {r0-r3, r12} 			/* r0-r3,r12 出栈 */
	pop {lr} 					/* lr 出栈 */
	subs pc, lr, #4 			/* 将 lr-4 赋给 pc */

/* FIQ 中断 */
FIQ_Handler:

	ldr r0, =FIQ_Handler 
	bx r0
```

## 链接脚本

```sh
SECTIONS{
	. = 0X87800000;
	.text :
	{
		obj/start.o 
		*(.text)
	}
	.rodata ALIGN(4) : {*(.rodata*)} 
	.data ALIGN(4) : { *(.data) } 
	__bss_start = .; 
	.bss ALIGN(4) : { *(.bss) *(COMMON) } 
	__bss_end = .;
}
```

## C语言部分



