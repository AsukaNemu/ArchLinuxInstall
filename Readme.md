# ArchLiunx

- ## Step 1

  1. #### 战前准备

     - iso镜像文件 

     - 存储介质
       - u盘、磁盘、虚拟机
     - 一份详尽的教程

  2. #### 资源搜集

     1. [重庆大学镜像站](https://mirrors.cqu.edu.cn/archlinux/iso/2026.01.01/)
     2. [非常好的教程，本文应该会跟着他的脚步写](https://www.bilibili.com/video/BV19DBqB4EY4/?spm_id_from=333.337.search-card.all.click&vd_source=9eca4fd36d66d0809b6110dfe44e24f1)

  3. #### 开工

- ## Step2

  - 确定自己的启动模式

    ​	一般情况下，虚拟机默认**bios** 启动，其他默认 **UEFI** 引导

    > [!NOTE] 
    >
    > 后文会根据启动项的不同要求制作不同格式的磁盘分区

  - 开机准备
    1. ### 首先连接网络

       > [!TIP]
       >
       > ~~如果字体小可以先设置字体~~
       >
       > 使用 `setfont ter-v32n`设置字体

       ```
       ip -a        #列出当前的连接信息
       ```

       有线网络自动连接，无线网需要使用 **iwd** 提供的命令行工具

       <!--由于我这里是虚拟机，所以无线命令以后施工-->

       连接完毕后使用 `ping` 命令测试网络是否连通

       有网之后系统后台会自动激活网络时间协议**NTP**，把时间同步到UTC事件时间

       如果不放心可以使用

       `timedatectl`来确认是否开启了**NTP**

       ```
       如果没有开启 NTP
       即 NTP service：no
       则使用
       timedatectl set-ntp true
       来开启
       ```

    2. ### 配置相关镜像源

       ```
       reflector -a 12 -c cn -f 10 --sort score --v --save /etc/pacman.d/mirrirlist
       -a 12  选项指定最近12小时更新过的源
       -c cn  指定所在的国家或者地区
       -f	   挑出最快的十个
       --sort 按照同步时间和下载速度综合评分进行排序
       --v    显示过程
       --save 将结果保存到这个文件
       ```

       接下来我们需要更新数据库并且安装密钥

       ```
       pacman —Sy archlinux-keyring
       -S 表示安装
       y  表示更新本地软件数据列表
       ```

       接下来 `回车`确认安装

       - 可选

         方便图形化显示界面

         我们可以安装一个终端文档管理器：**yazi**

         ```
         pacman -S yazi
         然后用yazi命令打开yazi
         yazi			#不要输入多余的指令
         ```

    3. ### 硬盘分区

       ```
       lsblk -pf		#列出当前分区情况
       -p		#列出完整设备名
       -f		#显示更多信息
       ```

       在里面找到自己要使用的硬盘

       如果你不确定自己找到了

       ```
       fdisk -l /dev/sda		#可以列出详细信息
       ```

       如果出现了**Microsoft** 或者**Windows**字样

       说明不是你要使用的硬盘，换一块一次找下去

       找到后使用

       ```
       cfdisk /dev/sda		#分区工具
       ```

       进行分区

       *在第一次进入空盘时会让你选择分区类型，**UEFI**引导请使用**gpt**分区*

       如果分区类型选择错了

       ```
       可以使用
       fdisk /dev/sda		#跟上你要编辑的硬盘
       ```

       输入**g**创建**gpt分区**

       再输入**w**保存更改

       

       接下来分两个盘，一个用来引导`100M`，一个用来存放 `剩余空间`

       同时修改引导盘的Type为**EFI System**

       引导盘上放esp文件

       <u>影响esp挂载点的是跟分区的文件系统</u>
       <u>文件系统决定了文件的存储和检索方式</u>
       <u>最常用的是EXT4和BTRFS</u>
       <u>不同的文件系统有不同的特性</u>
       <u>EXT4是文件系统界实用可靠的OG</u>
       <u>但对于arch linux这样的滚动发行版来说</u>
       <u>BTRFS是更好的选择</u>
       <u>BTRFS最大的特点是快照</u>
       <u>你可以把它理解为游戏里的存档和回档</u>
       <u>快照不止可以帮你恢复系统</u>
       <u>想象一下你在游戏里存档的目的</u>
       <u>除了以防万一</u>
       <u>还有什么</u>
       <u>没错作死哎</u>

       分区结束后点击 **Write**保存 键入 `yes`确认

       之后 **Quit**退出

    4. #### 格式化分区

       接下来要通过格式化分区，建立我们需要的文件系统

       <u>注意格式化的时候一定要确认设备名没有输错</u>

       ```
       mkfs.fat -F 32 /dev/sda1		#把ESP格式化为FAT32
       ```

       ```
       mkfs.btrfs /dev/sda2		#把根分区格式化为BTRFS
       ```

       

    5. ### 创建子卷

       - 它的作用之一是设置快照范围

       ```
       mount -t btrfs /dev/sda2 /mnt	#把根目录挂载到mnt目录
       -t 指Type
       btrfs 指文件系统是btrfs
       ```

       接着用BTRFS管理工具创建root子卷

       ```
       btrfs subvolume create /mnt/@
       命令		子卷		创建
       再创建一个@home的子卷
       btrfs subvolume create /mnt/@home
       ```

       *这里没有配置Swap（交换空间）*

       *实际上可以用把内存中的一部分空间用来交换，这样速度更快，还不会影响硬盘寿命*

       *虚拟内存功能可以通过内存压缩技术来弥补*

       *在后面我会介绍用ZERRUN把内存的一部分，当做交换空间的配置方法*

       ```
       lsblk -pf		#查看硬盘分区情况
       ```

       现在是根分区挂在到了**mnt**

       为了把root子卷挂载到mnt

       ```
       我们需要先取消挂载
       umount /mnt
       再把子卷挂载到mnt
       mount -t btrfs -o subvol=/@,compress=zstd /dev/sda2 /mnt
       subvol指定子卷
       compress指定透明压缩
       zstd是压缩算法
       ```

       透明压缩是BTLFS的另一个功能，在数据写入硬盘之前先进行压缩，可以提高硬盘的读写性能，节省空间，可以在后面加上冒号，指定压缩等级（1-15）

       ```
       mount --mkdir -t btrfs -o subvol=/@home,compress=zstd /dev/sda2 /mnt/home
       因为mnt里面没有home目录所以加上--mkdir
       ```

       最后挂载**esp**

       ```
       mount --mkdir /dev/sda1 /mnt/efi
       ```

- # Step3

  - ##### 正式安装系统

    ``` 
    pacstrap -K /mnt base base-devel linux(-zen) linux-firmware btrfs-progs
    		复制密钥  基本包 编译aur用  主线内核(性能特调) 基本固件    btrfs管理工具
    ```

    如果你是**marvell**网卡的话加上`linux-firmware-marvell`

    ------

    然后我们还需要安装一些最基本的功能性软件,首先是联网的工具,主流的桌面环境都默认使用network manager,想要完整的桌面体验就装这个

    ~~*当然你也可以选择安装我们之前用过的IWD*~~

    > [!IMPORTANT]
    >
    > **但是一定要装联网工具！！！**

    ```
    pacstrap -K /mnt  networkmanager vim sudo amd-ucode
    										intel把amd改成intel
    ```

    在切换进系统前

    ```
    我们需要用这段命令生成fs tab文件
    genfstab -U /mnt > /mnt/etc/fstab
    -U使用UUID指定分区
    ```

    接下来使用

    `arch-chroot /mnt`

  - ##### 进入系统

    我们需要做一些初期配置，先来设置时区

    ```
    ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    ```

    除了手动创建链接，我们还可以用

    ```
    timedatectl set-timezone Asia/Shanghai #来设置时区
    ```

    接着运行

    ```
    hwclock --systohc
    ```

    etc目录下会出现adjtime文件,它是系统用来调整时间误差的文件

  - ##### 系统的本地化设置

    ```
    vim /etc/locale.gen
    ```

    删除en_US 和第一个zn_CN前面的第一个#号然后我们要进行系统的本地化设置

    ```
    locale-gen	#生成本地化文件
    ```

    接着编辑**locale.conf**设置本地化

    ```
    vim /etc/locale.conf
    ```

    第一行输入 `LANG=en_US.UTF-8`保存退出

    ```
    设置主机名
    vim /etc/hostname
    挑一个你喜欢的输入第一行
    ```

    ```
    设置root账户密码
    passwd
    输入两次
    密码在/etc/shadow里，可以用cat打印出来
    ```

  - 安装Bootloader引导程序

    ```
    pacman -S grub efibootmgr
    
    grub-install --target=x86_64-efi --efi-directory=/efi --boot-directory=/efi --bootloader-id=eris
    有些主板识别不了EFI/BOOT/BOOTx64.efi以外的引导文件，这时候在指令末加上--removable就可以解决
    ```

  - 安装软件

    由于大部分软件会默认grab的安装位置，在boot目录下

    所以我们需要在grab的默认位置创建一个链接，指向efi/grab

    ```
    ln -s /efi/grub /boot/grub
    ```

    接下来要生成grab的配置文件

    ```
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

    这套命令会扫描系统,生成具体的启动项和启动流程,结果会打印在终端

    加上-o选项可以把结果保存到文件

    - *可选项：假如你是双系统*

      ```
      pacman -S os-prober exfat-utils
      ```

      前者搜索其他系统，后者找到windows的EFI分区

      ```
      os-prober
      ```

      运行以上命令你就可以找到你的windows

      接下来我们要编辑/etc/default/grub

      ```
      vim /etc/default/grub
      
      取消注释掉最后一行
      取消注释GRUB_SAVEDEFAULT=y
      GRUB_DEFUALT=0  =>  saved
      GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"  =>  "loglevel=5"
      
      完成以上操作后再次运行
      grub-mkconfig -o /boot/grub/grub.cfg
      ```

  - #### 内存压缩

    ```
    pacman -S zram-generator
    ```

    然后编辑它的配置文件

    启动并配置他的大小

    ```
    vim /etc/systemd/zram-generator.conf
    ```

    ```
    写入以下内容
    [zram0]
    zram-size = ram
    compression-algorithm = zstd
    ```

    ```
    更改grub文件
    vim /etc/default/grub
    
    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"  =>  "loglevel=5 zswap.enable=0"
    
    每一次编辑完grub的源文件之后,都记得要重新生成grub点CFG让修改生效
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

- # Step4

  - 你完成了所有配置

    接下来重启你的设备

    ```
    exit	#退出chroot环境
    reboot	#重启
    ```

    此时会自动卸载所有的挂载

  - 重启以后输入root账户

    键入对应密码

    ```
    systemctl enable --now NetworkManager #打开你的网络服务
    ```

    `nmtui`打开链接网络的TUI界面

    ```
    pacman -S fastfetch cmatrix
    ```

    

​	
