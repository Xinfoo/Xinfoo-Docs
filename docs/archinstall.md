# 为什么会有这篇教程

**Arch Linux** 以其极高的自定义性和简易的包管理吸引了不少用户，但是这也带来了一个问题，就是不太友好的安装过程。虽然可以用官方的 `archinstall` 但是这很容易让初学者之后面对一些基本问题束手无策。想自己安装？按照官方的教程装完了，开机就一个大黑屏终端，不知道干什么。所以这个教程就诞生了，这个教程主要在官方安装教程的基础上添加了关于显卡驱动，桌面环境等等这些开箱即用的配置。

!!! info "内容来源："
    本文基于 Arch Wiki ，所有内容均可在 Arch Wiki 上溯源，文章末尾放有参考链接。

!!! tip "提示："
    本文主要使用 vim 编辑器，在使用编辑器的段落可以替换为自己习惯用的编辑器。

!!! warning "本文档不适合小白："
    本文档只适用于那些会装系统，会装 Windows 以及装过其他开箱即用的发行版如 Ubuntu ，但是现在想要进阶的用户。如果您是一个完全没有装过系统，也不了解 Linux 基本的文件树，也不知道磁盘分区是个什么东西的人，那请先去补课再回看本教程。

---

## 确保您能启动 Arch Linux Live CD

您需要先下载 Arch Linux Live CD ，还需要一个能启动 Arch Linux Live CD 的工具，比如一个U盘。之后您要使用镜像写入工具比如 k3b 或者使用能直接启动镜像的工具比如 Ventoy ，具体准备过程在此不再赘述，下面的教程从您启动 Live CD 之后，进入 Shell 开始。

---

## 验证引导模式

运行：
```shell
cat /sys/firmware/efi/fw_platform_size
```

- 如果命令结果为 64，则系统是以 UEFI 模式引导且使用 64 位 x64 UEFI。

- 如果命令结果为 32，则系统是以 UEFI 模式引导且使用 32 位 IA32 UEFI。

- 如果命令结果为 No such file or directory ，则系统可能是以 BIOS 模式（或 CSM 模式，这两种模式通常出现在老旧的电脑或未经配置的虚拟机上）引导。

如果系统没有以您想要的模式（UEFI 或 BIOS）引导启动，请您参考自己的计算机或主板说明书。

---

## 检查网络和时间

### 1、检查网络

确保系统已经列出并启用了网络接口，运行：

```shell
ip link
```

**有线网络**在此时一般会自动连接，默认情况下，安装映像在启动时已经预先配置好并启用了 systemd-networkd、systemd-resolved ，在您的主机的 LAN 口接上网线的情况下，它们会自动配置连接。

**WiFi** ，您需要使用 iwctl 连接您的 WiFi ，iwctl 命令由 iwd 提供，这时要连接到WiFi，运行：

```shell
iwctl
```

这时会进入一个交互式命令提示符，要查看您的 WiFi 网卡的名字，输入：

```shell
device list
```

如果您的 WiFi 网卡已关闭，请将其打开，输入：

```shell
device (您要打开的网卡) set-property Powered on
```

之后扫描附近的 WiFi ，输入：

```shell
station (刚才打开的网卡) scan
```

再然后，就可以列出所有可用的 WiFi ，输入：

```shell
station (刚才使用的网卡) get-networks
```

最后，连接到您的 WiFi ，输入：

```shell
station (刚才使用的网卡) connect (列出的您的 WiFi 名字)
```

之后会提示您输入密码，输入正确的密码后，就可以连接了，然后退出，输入：

```shell
exit
```

**检查连接**，在配置完任意一种网络后都要检查网络是否成功连通，运行：

```shell
ping -c 4 archlinux.org
```

如果命令提示正常接收服务器返回的4个包，则网络正常，可以继续。

---

### 2、检查时间

为确保软件包签名校验成功以及防止 TLS 证书错误，Live 系统需要准确的时间，为此 systemd-timesyncd (一个时间同步服务)默认启用，也就是说当系统已经创建互联网连接后，系统时间将自动同步。但以防万一，依然要手动检查。  

运行：

```shell
timedatectl
```

请检查输出的三个时间是否同步，同步则继续。（在未修改任何配置的情况下， Live CD 使用的是 UTC 时间，如果您不在欧洲，本地时间和这三个显示的时间不同是正常现象，比如您在中国，您的时间应该是 UTC 时间再加8小时。）

---

## 磁盘分区与格式化

### 1、检查磁盘

系统如果识别到计算机的硬盘、U盘或者移动硬盘等，就会将其分配为一个块设备，如 `/dev/sda` 、`/dev/nvme0n1` 。一般情况下，使用 SATA 接口的硬盘显示为 `/dev/sdX` ，使用 NVMe 接口的硬盘显示为 `/dev/nvmeXnY` ，可以使用 lsblk 或者 fdisk 等工具查看，这里使用 fdisk 。  

运行：

```shell
fdisk -l
```

结果中以 `rom`、`loop` 或者 `airootfs` 结尾的设备可以被忽略。结果中以 `rpmb`、`boot0` 或者 `boot1` 结尾的 `mmcblk*` 设备也可以被忽略。  

如果您的磁盘有错误的话，命令应该会返回部分红字，没有的话则继续。

如果您在重装 Arch Linux 之前并没有完全删除磁盘分区，要全新安装清除磁盘的话，可以这样操作。

对于 SSD ，运行：

```shell
blkdiscard -f /dev/(要清除的磁盘)
```

对于 HDD ，运行：

```shell
sgdisk --zap-all /dev/(要清除的磁盘)
```

!!! danger "注意！"
    以上操作会完全清除数据，可能会造成不可挽回的损失，请谨慎操作！

---

### 2、分区
对于一个选定的要分区的硬盘，以下分区是必需的：

- 一个根分区（挂载在根目录） `/`

- 要在 UEFI 模式中启动，还需要一个 EFI 系统分区 `/boot`

这时要使用分区工具（fdisk 、parted、cfdisk 等等）修改分区表。例如使用 cfdisk 。  

运行：

```shell
cfdisk /dev/(要被分区的磁盘)
```

之后会提示您新建分区表，如果是 BIOS 引导的话请选择 dos ，如果是 UEFI 引导的话请选择 GPT 。
cfdisk 是一个易用的类图形化的磁盘分区工具，使用上下左右键移动光标，按照提示操作。如果您有使用 DiskGenius 等工具分区的经验的话，应该很轻易就能上手，如果不会使用，请查阅资料。  

对于 UEFI 和 GUID 分区表的分区实例：

| 已安装系统上的挂载点 | 分区设备 | 分区类型 | 注释 | 建议大小 |
| ---- | ---- | ---- | ---- | ---- |
| /boot | /dev/efi_system_partition | EFI System | EFI 系统分区 | 1GiB |
| [SWAP] | /dev/swap_partition | Linux swap | 交换空间 | 根据内存大小 |
| / | /dev/root_partition | Linux Data Partition | 根目录 | 磁盘的剩余空间 |

对于 BIOS 和 MBR 分区表的实例：

| 已安装系统上的挂载点 | 分区设备 | 分区类型 | 注释 | 建议大小 |
| ----| ---- | ---- | ---- | ---- |
| [SWAP] | /dev/swap_partition | Linux swap | 交换空间 | 根据内存大小 |
| / | /dev/root_partition | Linux Data Partition | 根目录 | 磁盘的剩余空间 |

关于 SWAP 分区，这个分区主要是被系统休眠和虚拟内存功能使用的，它的大小由您的主机内存大小决定，以下是 Redhat 推荐的 SWAP 大小规则：

| 内存大小 | 需要休眠时 SWAP 分区大小 | 不需要休眠时 SWAP 分区大小 |
| ---- | ---- | ---- |
| 小于2GiB | 内存大小的3倍 | 内存大小的2倍 |
| 2 ~ 8GiB | 内存大小的2倍 | 与内存大小相等 |
| 8 ~ 64GiB | 内存大小的1.5倍 | 至少4GiB |
| 大于64GiB | 不推荐使用休眠 | 至少4GiB |

---

### 3、格式化分区

创建分区后，必须使用合适的**文件系统**对每个新创建的分区进行格式化。

对于根分区，可以格式化为以下几个文件系统。

要格式化为 Ext4 文件系统，运行：

```shell
mkfs.ext4 /dev/(根分区)
```

要格式化为 XFS 文件系统，运行：

```shell
mkfs.xfs /dev/(根分区)
```

!!! tip "使用 F2FS 文件系统："
    我们注意到了 F2FS 文件系统在 SSD 和 叠瓦式机械硬盘上的优异性能，如果您使用这两种类型的硬盘作为根目录的话， F2FS 也是一个可选的选择。

如果您是 SSD ，用这条命令将根分区格式化为 F2FS 文件系统：

```shell
mkfs.f2fs -O extra_attr,inode_checksum,sb_checksum,compression /dev/(根分区)
```

如果您是叠瓦式机械硬盘，用这条命令将根分区格式化为 F2FS 文件系统：

```shell
mkfs.f2fs -m -O extra_attr,inode_checksum,sb_checksum,compression /dev/(根分区)
```

!!! warning "警告！"
    若运行的内核版本比创建 F2FS 文件系统的内核版本低，则文件系统可能无法使用。例如，使用 linux 包提供的内核创建文件系统，当系统需要降级到linux-lts 包提供的内核时，就可能出现问题。同时 F2FS 的 fsck 功能存在缺陷，容易造成数据损失，如果您的主机经常遭遇断电，不建议使用 F2FS 文件系统。

如果您是 UEFI 引导的话，这时要格式化 EFI 分区为 FAT32 文件系统：

```shell
mkfs.fat -F 32 /dev/(EFI 系统分区)
```

对于 SWAP 分区，格式化为 SWAP 文件系统：

```shell
mkswap /dev/(交换空间分区)
```

---

## 挂载磁盘分区

您需要将根分区挂载到 `/mnt` ，运行：

```shell
mount /dev/(根分区) /mnt
```

!!! tip "挂载 F2FS ："
    F2FS 文件系统并不是一个开箱即用的文件系统，挂载选项通常需要根据使用场景优化，在这里可以保持上面的挂载命令，但是也可以使用我们给出的更适合通用情况挂载选项 mount -o compress_algorithm=zstd:6,compress_chksum,atgc,gc_merge,lazytime /dev/(根分区) /mnt ，这些参数之后可以通过修改 fstab 继续优化。

然后使用 mkdir 在 /mnt 下创建任何剩余的挂载点（例如，为 /boot 而创建 /mnt/boot），并按相应的层级顺序挂载相应的磁盘卷。

使用 --mkdir 选项运行 mount 来创建指定的挂载点。或者，先使用 mkdir 创建挂载点再挂载。

!!! warning "注意！"
    挂载分区一定要遵循顺序，先挂载根（root）分区（到 /mnt），再挂载引导（boot）分区（到 /mnt/boot 或 /mnt/efi，如果单独分出来了的话），最后再挂载其他分区。否则您可能遇到安装完成后无法启动系统的问题。

对于 UEFI 系统，挂载 EFI 系统分区，运行：

```shell
mount --mkdir /dev/(EFI 系统分区) /mnt/boot
```

如果创建了交换空间卷，使用 swapon 启用它：

```shell
swapon /dev/(交换空间分区)
```

---

## 安装基本系统

### 1、选择镜像站

需安装的软件包会从文件 /etc/pacman.d/mirrorlist 中所列的镜像站下载。下载软件包时，列表中越靠前的镜像站会拥有越高的优先级。可以手动编辑文件，将离您所处地理位置最近的镜像移到文件的头部，例如，使用清华源。

用文本编辑器 vim 打开 mirrorlist 文件：

```shell
vim /etc/pacman.d/mirrorlist
```

之后按键盘上的上下左右键移动光标，按 i 键进入插入模式，在文件最顶端写入：

```text
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
```

之后按 esc 键退出插入模式，用键盘输入 : 符号进入 vim 的命令模式，输入 wq 加回车，保存并退出。如果写错了的话，在编辑模式按 x 键删除光标所在的文字，输入 q! 不保存退出。

更新软件包缓存以检查镜像站是否设置成功，运行：

```shell
pacman -Syy
```

如果速度足够快，而且没有报错，则设置正确。

---

### 2、安装必须的软件包

除 /etc/pacman.d/mirrorlist 之外的配置不会从 Live 环境传递到安装的系统中，所以要手动指定要安装的包。

例如：

```shell
pacstrap -K /mnt base base-devel linux-zen linux-zen-headers linux-firmware intel-ucode xfsprogs btrfs-progs exfatprogs dosfstools f2fs-tools ntfs-3g nano vi vim man-db man-pages texinfo
```

!!! warning "注意！"
    这条命令不能直接复制使用，请详细阅读下面的内容。

关于以上安装的这些包：

- base 这是安装 Arch Linux 必须装的包。

- base-devel 这是基本开发编译工具包，如果之后要使用 AUR ，那这个包是必须的。

- linux-zen、linux-zen-headers 内核和内核头文件，内核是必须的，头文件是编译一些驱动模块需要的包，建议安装。 Arch Linux 提供多种内核，这里使用了 zen 核，这是一个对桌面用户高性能的内核，可根据需要替换为其他内核和对应的头文件，具体请参阅 [Arch Wiki 的内核页面](https://wiki.archlinuxcn.org/wiki/内核) 中的说明。

- linux-firmware 驱动包，里面包含了多个驱动，为了避免您并非全部了解的主机上的其他设备无法工作，这个包必须安装。

- xfsprogs、btrfs-progs、exfatprogs、dosfstools、f2fs-tools、ntfs-3g 这些是几个常用文件系统的管理工具，有了这些您才能在之后处理这些文件系统的分区格式化之类的操作，建议安装。

- intel-ucode 这是 CPU 微码包，如果您的主机使用的是 AMD CPU ，请换成 amd-ucode ，如果您是虚拟机，则不需要这个包。

- nano、vi、vim 这是三个使用广泛的，在其他发行版普遍预装的文本编辑器，建议安装。如果您不想全部保留的话，必须保留 vi ，否则 visudo 无法工作 。

- man-db、man-pages、texinfo 手册页查看工具，输入 man <对应的命令> 即可查看该命令的用法，建议安装。

!!! info "信息："
    Arch Linux 官方的教程在这里推荐要安装网络管理软件，本教程将在进入 chroot 环境后配置，如果您在这一步之后不再继续看本教程，直接按照官方教程配置完重启了的话，您的主机是无法连接网络的。

---

## 配置系统

### 1、生成 fstab 文件

生成 fstab 文件以使需要的文件系统（如启动目录 /boot）在启动时被自动挂载，用 -U 设置 UUID 作为分区唯一标识符：

```shell
genfstab -U /mnt >> /mnt/etc/fstab
```

**强烈建议**在执行完以上命令后，检查一下生成的 /mnt/etc/fstab 文件是否正确。如果有问题，最好在现在手动修改。这个文件至关重要，它负责管理之后您的系统在启动时自动挂载分区的配置，如果异常，严重时，系统将无法正常启动。

简单的检查 fstab 文件：

```shell
cat /mnt/etc/fstab
```

在这里要尤其注意这个文件是否存在，若不存在，请尝试重新生成。

---

### 2、chroot 到新安装的系统

接下来的步骤需要像启动到新安装的系统一样直接与其环境、工具和配置进行交互，请 chroot 到新安装的系统：

```shell
arch-chroot /mnt
```

!!! info "注意："
    由于部分 systemd 工具如 hostnamectl、localectl 和 timedatectl 需要有效的 dbus 连接，因此无法在 chroot 内使用这些工具。

!!! warning "提示："
    此处使用的是 arch-chroot 而不是直接使用 chroot，注意不要输错了。

---

### 3、设置时间和时区

为顺应人类习惯（如显示正确的当地时间及处理夏令时），请设置时区：

```shell
ln -sf /usr/share/zoneinfo/地区名/城市名 /etc/localtime
```

!!! info "提示："
    例如，在中国大陆需要将时区设置为北京时间，那么请运行 ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime 。时区名称是上海而非北京，是因为上海是该时区内人口最多的城市。

然后运行 hwclock 以生成 `/etc/adjtime` ：

```shell
hwclock --systohc
```

这个命令假定已设置硬件时间为 UTC 时间，您的主机 BIOS 中显示的会是 UTC 时间，可能不是您当地的时间。

---

### 4、区域和本地化设置

程序和库如果需要本地化文本，都依赖区域设置，后者明确规定了地域、货币、时区、日期的格式、字符排列方式和其他本地化标准。需要设置这两个文件：locale.gen 与 locale.conf 。

编辑 /etc/locale.gen，然后取消掉 en_US.UTF-8 UTF-8 和其他需要的 UTF-8 区域设置前的注释（#）。

使用 vim 编辑 locale.gen ：

```shell
vim /etc/locale.gen
```

取消以下内容前的注释(#)：

```text
en_US.UTF-8 UTF-8
...
zh_CN.UTF-8 UTF-8
```

然后运行 locale-gen 以生成 locale 信息：

```shell
locale-gen
```

随后创建 locale.conf 文件，并编辑设定 LANG 变量，比如使用 vim 创建并编辑：

```shell
vim /etc/locale.conf
```

可以输入以下内容：

```text
LANG=en_US.UTF-8
```

!!! info "注意："
    如果在此处设置为中文 locale，比如 LANG=zh_CN.UTF-8 ，这可能会导致 tty 上中文显示为方块（因为 TTY 下没有 CJK 字体）。我们推荐在这里保持 locale 为英文，之后在桌面环境中设置中文即可。

---

### 5、网络配置

首先，我们需要安装网络管理工具，对于常用桌面环境的情况，我们推荐使用 NetworkManager 。
安装：

```shell
pacman -S networkmanager
```

如果您使用 WiFi 的话， iwd 通常是比 NetworkManager 自带的 WiFi 后端性能更好的选择。当然， iwd 是可以作为 NetworkManager 的默认 WiFi 后端的，可以这样操作。

安装 iwd ：

```shell
pacman -S iwd
```

之后，创建以下文件，使用 vim 创建并编辑：

```shell
vim /etc/NetworkManager/conf.d/wifi_backend.conf
```

输入以下内容：

```text
[device]
wifi.backend=iwd
```

这时就成功使用 iwd 作为后端了。

安装完网络管理器之后，我们要启用它：

```shell
systemctl enable NetworkManager.service
```

!!! warning "注意！"
    如果您使用 WiFi 并按照刚才的设置把 iwd 作为 WiFi 后端的话，这里不需要启动 iwd 的 systemd 单元， iwd 会作为 NetworkManager 的子进程由 NetworkManager 启动。

现在，我们要设置主机名。为您的系统设置一个固定且易于辨识的名称（在联网环境中尤其有用），请创建 hostname 文件，使用 vim 创建并编辑：

```shell
vim /etc/hostname
```

输入以下内容：

```text
(您的主机名)
```

!!! tip "提示："
    请参阅 [RFC 1178](https://datatracker.ietf.org/doc/html/rfc1178) 以获取一些关于为计算机取名的建议。如 hostname 所述，其必须包含 1-63 个字符，仅使用小写的 a-z、0-9 以及 -，但不得以 - 开头。

设置 hosts 文件，使用 vim 编辑 `/etc/hosts` ：

```shell
vim /etc/hosts
```

将内容更改为以下示例：

```text
127.0.0.1        localhost
::1              localhost
127.0.0.1        (您的主机名).localdomain (您的主机名)
```

网络配置完毕。

---

### 6、安装显卡驱动

对于使用 Intel 显卡和 AMD 以及旧的 ATI 显卡用户来说，以上这些显卡的驱动都包含在 linux-firmware 包中，这时通常是开箱即用的，不需要手动配置，这些显卡的用户跳过这一章节就好，继续下一章。在这里着重讲解一下 NVIDIA 显卡驱动。

首先，如果您不知道您的显卡代号或者型号的话，用以下命令检测：

```shell
lspci -k -d ::03xx
```

在 Arch Linux 上，不同系列的显卡有不同的 NVIDIA 驱动包支持，以下列出列表，表格从下到上代表着发布时间从早到晚，通过列表上的超链接得知您要安装的驱动型号：

| GPU 家族 | 适配驱动包 |
| ---- | ---- |
| Blackwell 架构，包括 H100 、H200 等最新的数据中心 GPU ，以及 RTX50 系列显卡 | nvidia-open-dkms 、 nvidia-open 、 nvidia-open-lts |
| [图灵架构](https://nouveau.freedesktop.org/CodeNames.html#NV160)、[安培架构](https://nouveau.freedesktop.org/CodeNames.html#NV170)和[艾达·洛芙莱斯架构](https://nouveau.freedesktop.org/CodeNames.html#NV190)的显卡 | nvidia-open-dkms 、 nvidia-open 、 nvidia-open-lts 、 [nvidia-580xx-dkms](https://aur.archlinux.org/packages/nvidia-580xx-dkms)(AUR) |
| [麦克斯韦架构](https://nouveau.freedesktop.org/CodeNames.html#NV110)、[帕斯卡架构](https://nouveau.freedesktop.org/CodeNames.html#NV130)和[伏特架构](https://nouveau.freedesktop.org/CodeNames.html#NV140)的显卡 | [nvidia-580xx-dkms](https://aur.archlinux.org/packages/nvidia-580xx-dkms)(AUR) |
| [开普勒架构](https://nouveau.freedesktop.org/CodeNames.html#NVE0)的显卡 | [nvidia-470xx-dkms](https://aur.archlinux.org/packages/nvidia-470xx-dkms)(AUR) |
| [费米架构](https://nouveau.freedesktop.org/CodeNames.html#NVC0)的显卡 | [nvidia-390xx-dkms](https://aur.archlinux.org/packages/nvidia-390xx-dkms)(AUR) |
| [特斯拉架构](https://nouveau.freedesktop.org/CodeNames.html#NV50)的显卡 | [nvidia-340xx-dkms](https://aur.archlinux.org/packages/nvidia-340xx-dkms)(AUR) |
| [居里架构](https://nouveau.freedesktop.org/CodeNames.html#NV40)以及更早的系列显卡 | 不受支持 |

如果您的显卡是最新推出的，但是又找不到在上面的支持驱动，您也许需要使用 [nvidia-open-beta](https://aur.archlinux.org/packages/nvidia-open-beta/)(AUR) 以获得更新版本的驱动。

!!! info "关于驱动包名："
    nvidia-open 包支持的是官方的 linux 内核包， nvidia-open-lts 同理，支持的是官方的 linux-lts 内核包，这两个包不需要内核头文件如 linux-headers 就可以安装运行，因为其驱动内核模块是提前编译好的。对于其他内核比如 linux-zen 或者其他自定义内核，需要使用 nvidia-open-dkms 或者其他后缀为 dkms 的包，这些包不包含编译好的内核模块，需要在您的主机上进行编译，所以需要对应安装的内核的头文件作为依赖，比如想在使用 linux-zen 内核的系统上安装 nvidia-open-dkms （或者其他后缀为 dkms 的包）作为驱动，则必须安装 linux-zen-headers 包。

!!! warning "使用旧显卡的用户注意："
    对于使用开普勒架构以及之前的任意架构的显卡的用户，我们不推荐使用 NVIDIA 推出的闭源驱动，因为对于这些显卡的驱动早就停更了，这些驱动并不支持 Wayland ，而且也不支持最新版的 X.org ，安装它们并不能获得好的体验，这里我们推荐使用开源内核模块 nouveau ，这个内核模块是内核自带的，而且开箱即用，如果您要使用 nouveau 的话，请直接跳过安装显卡驱动这一章节，继续下一章。

!!! warning "关于带有(AUR)后缀的包的安装："
    由于在 chroot 环境是直接使用 root 用户的，但是 Arch Linux 为了安全考虑， makepkg 是不支持以 root 身份运行的。如果您现在按照教程的顺序到了这一步的话，您现在应该在 chroot 的 root 环境，这时要跳过显卡驱动安装这一节，直接进行下一节的操作，之后在后面所有的流程结束重启后再回看本节进行操作。

安装任意 NVIDIA 的私有驱动包之前，我们要禁用内核自带的 nouveau 模块，以防止冲突，使用 vim 编辑 `/etc/mkinitcpio.conf` ：

```shell
vim /etc/mkinitcpio.conf
```

随后我们找到 HOOKS=(...) 这一行，将其中的 kms 删除，之后再进行安装驱动包。

在以上表格列出的驱动包中，带有(AUR)后缀的包的安装流程是与其他包不同的，这里分开说明。

**安装普通驱动包**  
这里我们使用的是 linux-zen 内核，以 nvidia-open-dkms 为例，运行：

```shell
pacman -S nvidia-open-dkms nvidia-utils nvidia-settings
```

驱动包安装完成。如需安装其他驱动包，如 linux 包对应的 nvidia-open ，直接用 nvidia-open 替换上面命令的 nvidia-open-dkms 即可。

**安装带有(AUR)后缀的驱动包**  
请确保您现在登录的不是 root 用户，而是您自己创建的那个账户，我们以安装 nvidia-580xx-dkms 为例，如需安装其他的 AUR 仓库的驱动包的话，直接用上表给出的包名 -dkms 前的内容替换掉命令中的 nvidia-580xx-utils 中的 nvidia-580xx 即可。

首先，我们要安装 git 以便之后克隆 AUR 包的仓库，安装 git ：

```shell
pacman -S git
```

安装完成后，使用 git 克隆驱动包的 AUR 仓库地址：

```shell
git clone https://aur.archlinux.org/nvidia-580xx-utils.git
```

随后，我们切换到包仓库目录：

```shell
cd nvidia-580xx-utils
```

之后运行：

```shell
makepkg -sir
```

在运行时按提示操作，完成之后，安装其他工具，运行：

```shell
pacman -S nvidia-settings
```

驱动包安装完成。

!!! info "关于 mkinitcpio ："
    Arch Linux 官方的安装教程提醒在删除 `/etc/mkinitcpio.conf` 中的 kms 钩子之后，要重新运行 mkinitcpio -P ，但是据笔者观察，安装驱动包的时候实际上会自动运行这条命令，如果您在之后没有修改配置，比如 MODULES 段的早启动模块，那就不需要再次运行这条命令。

---

### 7、安装蓝牙驱动

如果您拥有 WiFi 无线网卡设备或者您的主机是笔记本，那一般自带蓝牙设备，这时候需要安装驱动，如果您使用的是纯台式机，没有蓝牙设备，那可以跳过本章节，继续下一章。

现在广泛使用的蓝牙协议栈实现是 BlueZ ，我们需要安装它：

```shell
pacman -S bluez bluez-utils
```

关于这些包：

- **bluez** 这个软件包提供蓝牙协议栈。

- **bluez-utils** 这个软件包提供 `bluetoothctl` 实用程序。

然后，我们启用蓝牙服务：

```shell
systemctl enable bluetooth.service
```

蓝牙配置完成。

---

### 8、安装桌面环境

在安装桌面环境之前，为了能让其显示中文，我们需要安装几款常用字体，运行：

```shell
pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra ttf-jetbrains-mono ttf-dejavu ttf-nerd-fonts-symbols ttf-nerd-fonts-symbols-mono
```

对于以上安装的包的解释：

- **noto-fonts 、 noto-fonts-cjk 、 noto-fonts-emoji 、 noto-fonts-extra** Google 的 Noto 字体家族，为 Android 操作系统默认字体，其中 noto-fonts-cjk 用于显示中日韩三国文字， noto-fonts-emoji 用于正确显示 emoji ，安装这些包基本可以保证桌面环境能够显示地球上所有在使用的文字。

- **ttf-jetbrains-mono 、 ttf-dejavu** 两款优秀的编程字体，可以优化终端显示。

- **ttf-nerd-fonts-symbols 、 ttf-nerd-fonts-symbols-mono** 用于显示某些终端程序上的图标，比如装了主题的 Zsh 或者 Neovim 。

桌面环境有非常多种，通常，我们的主机安装一个桌面环境即可，我们这里给出 KDE Plasma 和 GNOME 这两种桌面环境的安装。

**安装 KDE Plasma**  
KDE 是一套由桌面环境(KDE Plasma)、应用程序（KDE Applications）以及 Qt 附加库（KDE Frameworks）构成的软件项目。
要安装基本的 Plasma 桌面，运行：

```shell
pacman -S plasma
```

通常这还不够，我们还需要安装 KDE 的配套应用，要全部安装他们，运行：

```shell
pacman -S kde-applications
```

通常，我们并不需要安装完整的 KDE 应用包，这个软件包组数量非常庞大，有很多根本用不上的软件。这里给出我们推荐的安装的 KDE 桌面配套软件，运行：

```shell
pacman -S --needed konsole yakuake dolphin ark kate okular partitionmanager filelight gwenview k3b  kcalc kompare ksystemlog
```

对于以上安装软件包的解释：

- **konsole** KDE 默认的终端。

- **yakuake** 下拉式终端。

- **dolphin** 文件管理器。

- **ark** 压缩文件查看与解压工具。

- **kate** KDE 默认的文本编辑器。
- **okular** PDF 以及其他文档格式的简单查看器。

- **partitionmanager** 图形化的磁盘分区管理工具。

- **filelight** 图形化的磁盘占用情况查看工具。

- **gwenview** 图片浏览工具。

- **k3b** 光盘刻录以及 ISO 镜像生成工具。

- **kcalc** 计算器。

- **kompare** 文本对比工具。

- **ksystemlog** 系统日志查看器。

通过安装这些包，你通常能获得一个最小化的开箱即用的 KDE Plasma 桌面环境。

*安装中文输入法* 对于 KDE 来说，推荐的输入法是 Fcitx5 ，我们现在安装它，运行：

```shell
pacman -S fcitx5-im fcitx5-chinese-addons
```

设置输入法环境变量，使用 vim 编辑 `/etc/environment` ：

```shell
vim /etc/environment
```

将以下内容加入：

```text
XMODIFIERS=@im=fcitx
```

保存并退出，稍后在设置中的虚拟键盘选项，选中 Fcitx 即可启用它，输入法安装完成。

最后，启用登录管理器：

```shell
systemctl enable sddm.service
```

KDE Plasma 桌面环境安装完成。

**安装 GNOME 桌面环境**  
GNOME 是一个追求简单易用的桌面环境。它由 GNOME项目设计，并且完全由自由开源的软件组成。Ubuntu 等发行版使用的就是 GNOME 桌面环境，安装它可以实现从 Ubuntu 无缝迁移。

要安装它，有三个包组可以选择：

- **gnome** 包组，包含基本的桌面环境和一些集成良好的应用。

- **gnome-circle** 包组，包含多种格外应用，极大的拓展了Gnome生态。

- **gnome-extra** 包组，包含部分开发工具，以及其他适合Gnome的应用与游戏。

通常情况下，只安装 gnome 包组即可获得良好的体验，运行：

```shell
pacman -S gnome
```

由于 GNOME 移除了托盘显示功能，要使其他应用显示托盘，需要通过插件来实现，我们现在要安装它，运行：

```shell
pacman -S gnome-shell-extension-appindicator
```

在安装系统完成重启后即可在插件管理启用它。

对于 GNOME 桌面来说，还有许多额外应用值得安装，运行：

```shell
pacman -S gnome-tweaks dconf-editor
```

对于以上安装的软件包的说明：

- **gnome-tweaks** 设置 GNOME 的额外的高级功能，例如，全局缩放，切换字体等等。

- **dconf-editor** 直接操作 GNOME 的设置数据库，可以启用许多实验性功能。

*安装中文输入法* 对于 GNOME 桌面环境来说，推荐的输入法是 ibus ，我们现在要安装它，运行：

```shell
pacman -S ibus ibus-libpinyin
```

在安装系统完成后在设置应用里启用它即可，输入法安装完成。

最后，启用登录管理器：

```shell
systemctl enable gdm.service
```

GNOME 桌面环境安装完成。

---

### 9、配置时间同步

虽然有多个工具可以做到时钟同步，但是对于普通用户来说，最适合的工具还是 systemd-tiemesyncd ， systemd-timesyncd已经包含在 systemd 包中， systemd 已经随 base 包组安装，这时只需要直接启用它。

首先，编辑配置文件，使用 vim 编辑 `/etc/systemd/timesyncd.conf` ：

```shell
vim /etc/systemd/timesyncd.conf
```

在文件中找到 #NTP= 这一行，取消注释，并添加时间同步服务器，在中国，设置成这样即可：

```text
NTP=cn.ntp.org.cn time.windows.com cn.pool.ntp.org time.cloudflare.com
```

之后，启用 systemd 单元：

```shell
systemctl enable systemd-timesyncd.service
```

时间同步服务配置完成。

---

### 10、为 root 账户创建密码并创建非特权账户

为您的主机的 root 账户创建密码，运行：

```shell
passwd root
```

之后将提示输入您要设置的密码。

我们需要为主机创建非特权账户，日常使用 root 账户登录是十分危险的操作，应当创建非特权账户进行日常操作，仅在管理系统时使用 root 。要创建非特权账户，运行：

```shell
useradd -m -G wheel -s /bin/bash "(用户名)"
```

对于命令的解释：

- **-m** 选项是在此时为该用户创建在 `/home` 目录下的家目录，目录为 `/home/(用户名)` 。

- **-G** 选项是为该用户指定附加组，这里附加的是 `wheel` ，在 Arch Linux 中，这个组拥有 sudo 权限。

- **-s** 选项是指定该用户使用的 Shell ，选项之后追加要使用的 Shell 的可执行文件目录。

为新用户设置密码：

```shell
passwd (用户名)
```

之后要使用 sudo ，这时要为 wheel 组启用 sudo 权限，运行：

```shell
visudo
```

打开了一个和 vim 编辑器相同的界面，这里的操作方式和 vim 相同。我们要移动光标，找到以下内容并删除前面的注释：

```text
%wheel      ALL=(ALL:ALL) ALL
```

保存并退出后 sudo 权限配置成功。

账户设置完成。

---

### 11、安装系统引导

对于系统引导，有多个引导工具可供选择，这里我们使用 GRUB ，这是目前功能最强大，支持最丰富，也是最稳定的引导管理器。

!!! info "安全启动："
    我们这里不涉及配置 UEFI 下的安全启动，如需配置安全启动，请移步 Arch Wiki 上的 [GRUB](https://wiki.archlinuxcn.org/wiki/GRUB) 页面。

要使用 GRUB 引导，首先要安装它，运行：

```shell
pacman -S grub efibootmgr os-prober
```

关于上面安装的软件包：

- **grub** 引导管理器主程序。

- **efibootmgr** 用于创建 UEFI 启动项的工具，由它创建 UEFI 启动条目。

- **os-prober** 其他引导探测工具，用于探测其他系统的引导。

安装完 GRUB 包本体后，要为系统安装引导，分为 UEFI 引导和 BIOS 引导。

为使用 **UEFI** 的系统安装引导，运行：

```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id='Arch Linux'
```

!!! info "关于 --removable 选项："
    正常情况下，使用上面的那条命令一般能正常安装引导并启动，但是某些主板(尤其是 MSI 微星的主板)会检测 EFI 分区下的 BOOT 文件夹内的文件，如果 BOOTX64.EFI 不存在，则不会为硬盘创建启动条目，这时需要在上面那条命令的基础上加上 --removable 参数运行，如果你使用了 --removable 选项，那 GRUB 将被安装到 `esp/EFI/BOOT/BOOTX64.EFI` 这时可以正常启动。

我们推荐再用 --removable 选项安装一次引导，以作为备用，运行：

```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id='Arch Linux' --removable
```

对于 UEFI 系统的 GRUB 引导安装成功。

为使用 **BIOS** 引导的系统安装引导，运行：

```
grub-install --target=i386-pc /dev/(要安装引导的磁盘)
```

!!! info "注意："
    这里的 i386-pc 是有意为之，与你机器的实际架构无关，保持这样即可， 其中 /dev/(要安装的磁盘) 是要安装 GRUB 的磁盘(不是分区)，比如磁盘 `/dev/sda` 、 `/dev/nvme0n1` 或者 `/dev/mmcblk0` ，而不是分区 `/dev/sda1` 。

对于 BIOS 系统的 GRUB 引导安装成功。

**生成 GRUB 配置文件**，无论是 UEFI 引导或者 BIOS 引导都需要生成 GRUB 配置文件才能使用，运行：

```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

运行完之后，GRUB 引导安装完成。

---

### 12、退出 chroot 环境并卸载硬盘

输入：

```shell
exit
```

退出 chroot 环境。

卸载硬盘，输入：

```shell
umount -R /mnt
```

确保无误后，进行下一步。

---

## 重启系统并切换启动条目

输入：

```shell
reboot
```

以重启。

重启期间，请按快捷键（一般是Delete，具体请查阅您的主板说明书）进入 BIOS ，然后切换 Arch Linux 启动条目为第一启动项，具体操作流程请查阅您的主板说明书，之后重启系统。

重启之后，如果您根据之前显卡安装条目的建议，跳过了带有(AUR)后缀的驱动包安装，这时候，您可以返回条目并安装显卡驱动了。

---

## 您的 Arch Linux 安装完毕
如果你从头到尾未遇到不可解决的问题的话，我们在这里恭喜您，您的 Arch Linux 已经安装完成，让我们一起欢呼！！🌺🌺

关于之后要做的事情，我们给出 Arch Wiki上的建议。

[了解 Arch Linux ！](https://wiki.archlinuxcn.org/wiki/Arch_Linux)

[关于 Arch Linux 和其他发行版的区别](https://wiki.archlinuxcn.org/wiki/Arch_与其他发行版的比较)

[如何不弄坏你的 Arch Linux ？](https://wiki.archlinuxcn.org/wiki/建议阅读/给新用户的关于如何不去弄坏_Arch_Linux_系统的建议)

[建议阅读！管理您的 Arch Linux ！](https://wiki.archlinuxcn.org/wiki/建议阅读)

---

# 参考资料

https://wiki.archlinuxcn.org/wiki/安装指南

https://wiki.archlinuxcn.org/wiki/Iwd

https://wiki.archlinuxcn.org/wiki/F2FS

https://wiki.archlinuxcn.org/wiki/网络配置

https://wiki.archlinuxcn.org/wiki/NVIDIA

https://wiki.archlinuxcn.org/wiki/GRUB

https://wiki.archlinuxcn.org/wiki/KDE

https://wiki.archlinuxcn.org/wiki/GNOME

