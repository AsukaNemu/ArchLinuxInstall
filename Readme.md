# Arch Linux

本文记录 **x86_64 UEFI + GPT + Btrfs + GRUB** 的安装流程；BIOS/CSM 安装需另按官方指南选择引导方式。
设备名 `/dev/sda`、`/dev/sda1`、`/dev/sda2` 都是示例，执行分区或格式化前须按实际磁盘替换。

命令复核：2026-10-29。参考 [Arch 安装指南](https://wiki.archlinux.org/title/Installation_guide)、
[Reflector 手册](https://man.archlinux.org/man/reflector.1.en)、[Arch GRUB 文档](https://wiki.archlinux.org/title/GRUB)。
本次为文档复核，未完成真实硬盘或虚拟机的全流程安装验收。

- ## Step 1

  1. #### 战前准备

     - iso镜像文件 

     - 存储介质
       - u盘、磁盘、虚拟机
     - 一份详尽的教程

  2. #### 资源搜集

     1. [Arch 官方下载页（选择当前镜像并校验签名）](https://archlinux.org/download/)
     2. [非常好的教程，本文应该会跟着他的脚步写](https://www.bilibili.com/video/BV19DBqB4EY4/?spm_id_from=333.337.search-card.all.click&vd_source=9eca4fd36d66d0809b6110dfe44e24f1)

  3. #### 开工

- ## Step2

  - 确定自己的启动模式

    在安装环境中检查实际启动模式，不能按是否为虚拟机判断：

    ```bash
    cat /sys/firmware/efi/fw_platform_size
    ```

    返回 `64` 表示本文使用的 64 位 UEFI 环境；路径不存在通常表示 BIOS/CSM 模式，请先按官方指南处理。

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
       ip address   #列出当前的接口与地址
       ```

       有线网络自动连接，无线网需要使用 **iwd** 提供的命令行工具

       <!--由于我这里是虚拟机，所以无线命令以后施工-->

       连接完毕后使用 `ping` 命令测试网络是否连通

       有网之后安装环境会自动激活网络时间协议**NTP**，把时间同步到 UTC 时间

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
       reflector --country China --age 12 --protocol https --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
       --age 12    只保留最近 12 小时同步过的镜像
       --country   指定国家；可按所在地修改
       --protocol  使用 HTTPS
       --latest 10 从符合条件的镜像中选取最近同步的十个
       --sort rate 按下载速率排序
       --save 将结果保存到这个文件
       ```

       当前安装镜像通常已有可用密钥。只有安装环境遇到旧密钥问题时，才更新密钥包：

       ```
       pacman -Sy archlinux-keyring
       -S 表示安装
       y  表示更新本地软件数据列表
       ```

       此命令限安装介质中的临时环境；已安装的系统应使用 `pacman -Syu` 完整升级，避免部分升级。

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

       同时核对容量、型号与现有分区，明确哪些数据需要保留；不能只按是否出现 Windows 字样选择磁盘。
       双系统安装应保留 Windows 和现有 ESP，只在确定的可用空间上分区，不要对含有待保留数据的磁盘重新建表。

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

       仅对全新空盘，才输入 **g** 创建 **GPT 分区表**；这会替换原有分区表。

       再输入**w**保存更改

       

       全新空盘可创建一个约 `1 GiB` 的 EFI 系统分区（ESP），其余空间用于根分区。
       双系统如复用现有 ESP，先检查可用空间，**不要重新格式化它**。

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
       mkfs.fat -F 32 /dev/sda1		#仅格式化新建的ESP为FAT32；复用ESP时跳过
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
    pacstrap -K /mnt base base-devel linux linux-firmware btrfs-progs
    # -K 生成目标密钥；base 是基本包，base-devel 用于构建 AUR 包
    ```

    如选择 Zen 内核，将上方包名 `linux` 替换为 `linux-zen`；不要输入 `linux(-zen)`。

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
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    ```

    chroot 中使用上方符号链接设置时区；`timedatectl` 需要活动的 D-Bus，留到首次正常启动后使用。

    接着运行

    ```
    hwclock --systohc
    ```

    etc目录下会出现adjtime文件,它是系统用来调整时间误差的文件

  - ##### 系统的本地化设置

    ```
    vim /etc/locale.gen
    ```

    取消 `en_US.UTF-8 UTF-8` 和 `zh_CN.UTF-8 UTF-8` 前面的 `#`，然后生成本地化文件：

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
    ```

    在 chroot 中启用已安装的网络服务，确保首次启动可联网：

    ```bash
    systemctl enable NetworkManager.service
    ```

  - 安装Bootloader引导程序

    ```
    pacman -S grub efibootmgr
    
    grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=eris
    有些主板识别不了EFI/BOOT/BOOTx64.efi以外的引导文件，这时候在指令末加上--removable就可以解决
    ```

  - 安装软件

    上方命令将 EFI 程序安装到 `/efi`，GRUB 模块与配置仍放在默认 `/boot/grub`，无需额外创建符号链接。

    接下来生成 GRUB 的配置文件：

    ```
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

    这套命令会扫描系统,生成具体的启动项和启动流程,结果会打印在终端

    加上-o选项可以把结果保存到文件

    - *可选项：假如你是双系统*

      ```
      pacman -S os-prober
      ```

      `os-prober` 用于搜索其他系统。Windows 的 ESP 是 FAT 文件系统，不需要 exFAT 工具来检测。
      只有需要管理 exFAT 数据分区时，才另装当前官方包 [exfatprogs](https://archlinux.org/packages/extra/x86_64/exfatprogs/)：`pacman -S exfatprogs`。
      检测 UEFI Windows 前，先挂载包含 Windows Boot Manager 的 ESP；chroot 中未检出时可在首次启动后重试。

      ```
      os-prober
      ```

      运行以上命令你就可以找到你的windows

      接下来我们要编辑/etc/default/grub

      ```
      vim /etc/default/grub
      
      添加或取消注释 GRUB_DISABLE_OS_PROBER=false
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
    umount -R /mnt  #卸载安装目标
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

    

# 	 等待施工ing
