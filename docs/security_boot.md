# 为 Arch Linux 配置安全启动

我们要为 Arch Linux 配置安全启动，使用 systemd-boot 作为引导管理器。

## 安装工具

```shell
sudo pacman -S sbsigntools mokutil
```

## 创建一个保存证书的目录(请妥善保管该目录)，并安装由 Fedora 签名的 shim

```shell
mkdir Security_Boot
```

```shell
cd Security_Boot
```
```shell
git clone https://aur.archlinux.org/shim-signed.git
```

```shell
cd shim-signed
```
```shell
makepkg -sir
```

## 替换引导管理器

```shell
cp "/usr/share/shim-signed/shimx64.efi" "/boot/EFI/BOOT/BOOTX64.EFI"
```

```shell
cp "/usr/share/shim-signed/mmx64.efi" "/boot/EFI/BOOT/MMX64.EFI"
```

## 生成自己的证书

```shell
openssl req \
    -new \
    -x509 \
    -newkey rsa:4096 \
    -keyout MOK.key \
    -out MOK.crt \
    -nodes \
    -days 3650 \
    -subj "/CN=Custom/"
```
此处的 Custom 可被替换为任意内容。

## 转化证书

```shell
openssl x509 \
    -outform DER \
    -in MOK.crt \
    -out MOK.cer
```
## 导入证书并设置临时密码

```shell
sudo mokutil --import MOK.cer
```

重启电脑并输入刚才的临时密码
```shell
reboot
```

## 签名 systemd-boot 和 Linux Image

```shell
sbsign \
    --key MOK.key \
    --cert MOK.crt \
    --output systemd-bootx64.efi \
    /boot/EFI/systemd/systemd-bootx64.efi
```

```shell
sbsign \
    --key MOK.key \
    --cert MOK.crt \
    --output vmlinuz-linux-zen \
    /boot/vmlinuz-linux-zen
```
如果是其他内核请替换 vmlinuz-linux-zen。

## 复制 systemd-boot 和 Linux Linux Image

```shell
sudo cp systemd-bootx64.efi /boot/EFI/BOOT/GRUBX64.EFI
```

```shell
sudo cp vmlinuz-linux-zen /boot/vmlinuz-linux-zen
```
## 检查命令

- 检查证书是否导入
```shell
sbverify --list XX.der
```

- 检查启动状态
```shell
bootctl status
```

- 检查证书
```shell
sudo mokutil --list-enrolled
```

## 清除处于 BIOS 的 NVRAM 中的 MOK 证书

设置临时修改密码
```shell
sudo mokutil --reset
```

重启电脑
```shell
reboot
```
之后再Shim的界面输入密码移除所有密钥。
