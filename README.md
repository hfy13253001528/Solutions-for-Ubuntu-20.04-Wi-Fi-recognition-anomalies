# Solutions-for-Ubuntu-20.04-Wi-Fi-recognition-anomalies
rt
# Ubuntu 20.04（内核 5.15）修复 Realtek RTL8852CE WiFi

> 适用：Ubuntu 20.04 LTS（focal，内核 5.15 系列）
> 网卡：Realtek **RTL8852CE**（PCI ID `10ec:c852`）
> 目标：在不升级系统、不影响有线网络的前提下，恢复 WiFi。

---

## 1. 症状

- `lspci` 能看到无线网卡，但**没有任何内核驱动绑定**。
- `dmesg` / `lspci -k` 显示只有 `rtw89_core`、`rtw89_pci`，**没有 `rtw89_8852ce`**。
- 系统只能靠有线或手机 USB 共享上网。

快速确认：

```bash
lspci -nnk -s "$(lspci -nn | grep -i 'network controller' | cut -d' ' -f1)"
```

## 2. 根因

- 芯片是 **RTL8852CE**，主线的 `rtw89_8852ce` 驱动要 **内核 ≥ 6.2** 才有。
- Ubuntu 20.04 的 HWE 内核永远停在 **5.15**（实测连最新的 `linux-modules-extra-5.15.0-134` 里也**没有** 8852ce），所以**只能装外置驱动**。
- **千万别装错驱动**：
  - `10ec:c852` = **RTL8852CE**
  - `10ec:b852` = **RTL8852BE**
  - 两者驱动不通用，装 BE 驱动会因 PCI ID 不匹配而**不绑定**。

## 3. 解决：Realtek 官方驱动 + DKMS

### 3.0 确认芯片型号

```bash
lspci -nn | grep -i network
#   10ec:c852 -> RTL8852CE  （本文）
#   10ec:b852 -> RTL8852BE
```

### 3.1 安装依赖

```bash
sudo apt update
sudo apt install -y dkms build-essential linux-headers-$(uname -r) bc
```

### 3.2 下载 Realtek 官方 rtl8852CE 驱动源码（v1.19.4.4）

```bash
cd ~
curl -L -o rtl8852ce.tar.gz \
  "https://codeload.github.com/qianguangzhi/rtl8852CE_WiFi_linux_v1.19.4.4-0-g0b81da37b.20230906/tar.gz/refs/heads/main"
tar xzf rtl8852ce.tar.gz
```

### 3.3 放到 `/usr/src` 并写 `dkms.conf`

```bash
sudo rm -rf /usr/src/8852ce-1.19.4.4
sudo mkdir -p /usr/src/8852ce-1.19.4.4
sudo cp -a ~/rtl8852CE_WiFi_linux_v1.19.4.4-0-g0b81da37b.20230906-main/. /usr/src/8852ce-1.19.4.4/

sudo tee /usr/src/8852ce-1.19.4.4/dkms.conf >/dev/null <<'EOF'
PACKAGE_NAME="8852ce"
PACKAGE_VERSION="1.19.4.4"
BUILT_MODULE_NAME[0]="8852ce"
BUILT_MODULE_LOCATION[0]="."
DEST_MODULE_LOCATION[0]="/kernel/drivers/net/wireless/realtek/rtl8852ce"
MAKE[0]="sh -c 'make -j8 KVER=${kernelver} KSRC=/lib/modules/${kernelver}/build'"
CLEAN="make clean"
AUTOINSTALL="yes"
EOF
```

### 3.4 DKMS 编译安装

```bash
sudo dkms add     -m 8852ce -v 1.19.4.4
sudo dkms build   -m 8852ce -v 1.19.4.4
sudo dkms install -m 8852ce -v 1.19.4.4
```

### 3.5 加载模块

```bash
sudo modprobe cfg80211
sudo modprobe 8852ce
ip -br link          # 出现 wlan0 或 wlp8s0 即成功
```

## 4. 三个关键坑

1. **DKMS 会往 make 命令里注入 `KERNELRELEASE=`**
   DKMS 会把 `MAKE[0]` 中以 `make` 开头的部分替换成 `make -jN KERNELRELEASE=<内核>`，
   该变量透传到 kbuild 子 make，导致厂商 Makefile 走错分支、**编译出空模块**（日志里只有 MODPOST、没有 CC）。
   → 把 `MAKE[0]` 写成 `sh -c 'make ...'`，命令不以 `make` 开头，DKMS 就不注入。**这是本方案最核心的一步。**

2. **Secure Boot 必须关闭**（BIOS 里关，或自行签名），否则模块无法加载。

3. **不会影响有线网络**
   本操作只新增 `8852ce` 模块，有线 `r8169`（`enpX`）和手机共享 `usbnet/rndis_host`（`usb0`）完全不受影响。

## 5. 验证

```bash
dkms status                          # 8852ce, 1.19.4.4, <内核>, x86_64: installed
modinfo 8852ce | grep -E "filename|version"
iw dev wlp8s0 scan | grep SSID      # 能扫到热点
```

- 内核升级后，DKMS 因 `AUTOINSTALL="yes"` 会**自动重编**，WiFi 不会失效。
- 重启后 udev 会按 PCI 别名自动加载，无需手动 `modprobe`。

## 6. 卸载 / 回退

```bash
sudo dkms remove -m 8852ce -v 1.19.4.4 --all
sudo rm -rf /usr/src/8852ce-1.19.4.4
sudo rmmod 8852ce     # 当前会话立即卸载
```

## 7. 常见问题

| 现象 | 说明 |
| --- | --- |
| dmesg 报 `TXPWR_LMT.txt ... Fail` | 缺少可选的发射功率限制表，驱动用默认值，**无害** |
| dmesg 报 `compiler differs` | 编译用 gcc 与内核构建 gcc 小版本不同，**无害** |
| `insmod: Unknown symbol cfg80211_*` | 先 `sudo modprobe cfg80211` 再加载 |
| 蓝牙 `0bda:5852` 不可用 | 5.15 下通常能用；如需修复用 `HRex39/rtl8852be_bt`，并把 `0bda:5852` 加进 `btusb.c` |

## 8. 适用范围

- **本方案专治 Ubuntu 20.04 / 内核 5.15 + RTL8852CE。**
- 若系统内核 **≥ 6.2**（如 Ubuntu 22.04/24.04），内核自带 `rtw89_8852ce`，**无需本方案**。

---

参考：Realtek 官方 Linux 驱动 `rtl8852CE_WiFi_linux_v1.19.4.4`。
