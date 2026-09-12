# AIC8800D80 Linux USB Wi-Fi Driver

[中文](#中文说明) | [English](#english)

This repository contains the AIC8800D80 USB Wi-Fi driver, firmware, and Linux 6.8 compatibility fixes. It was built and tested on Ubuntu 22.04 with kernel `6.8.0-124-generic` and USB device `a69c:8d80` / `a69c:8d83`.

本仓库包含 AIC8800D80 USB Wi-Fi 驱动、固件及 Linux 6.8 兼容补丁。已在 Ubuntu 22.04、内核 `6.8.0-124-generic` 和 USB 设备 `a69c:8d80` / `a69c:8d83` 上完成编译及实机验证。

## 中文说明

### 已验证功能

- 固件从 `a69c:8d80` 启动态下载，并重新枚举为 `a69c:8d83`
- 2.4 GHz 和 5 GHz Wi-Fi 扫描
- WPA/WPA2 关联
- DHCP 获取 IPv4 地址
- NetworkManager 管理
- 模块卸载、重新加载和 USB 重新插入

### 支持环境

当前版本已验证：

- Ubuntu 22.04
- Linux `6.8.0-124-generic`
- x86_64
- AIC8800D80 USB Wi-Fi，VID:PID 为 `a69c:5721`、`a69c:8d80` 或 `a69c:8d83`

代码保留了旧内核兼容分支，但其他发行版及内核版本尚未逐一验证。

### 1. 安装编译依赖

```bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r) eject
```

确认当前内核的构建目录存在：

```bash
test -e /lib/modules/$(uname -r)/build && echo "kernel headers ready"
```

### 2. 安装固件和 udev 规则

在仓库根目录执行：

```bash
sudo install -d /lib/firmware/aic8800D80
sudo install -m 0644 aic8800_linux_drvier/fw/aic8800D80/* /lib/firmware/aic8800D80/
sudo install -m 0644 aic8800_linux_drvier/tools/aic.rules /etc/udev/rules.d/aic.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

udev 规则会弹出网卡自带的只读存储设备，使硬件从 `a69c:5721` 切换到 Wi-Fi 启动态。

### 3. 编译驱动

```bash
cd aic8800_linux_drvier/drivers/aic8800
make clean
make
```

成功后会生成：

```text
aic_load_fw/aic_load_fw.ko
aic8800_fdrv/aic8800_fdrv.ko
```

编译时可能看到 `no previous prototype` 警告。这些是原厂源码已有的非致命警告；最终构建应以退出状态 0 结束。

### 4. 安装并加载模块

仍在 `aic8800_linux_drvier/drivers/aic8800` 目录中执行：

```bash
sudo make install
sudo modprobe cfg80211
sudo modprobe aic_load_fw
sudo modprobe aic8800_fdrv
```

如果旧模块已经加载，先替换内存中的版本：

```bash
sudo modprobe -r aic8800_fdrv aic_load_fw
sudo modprobe aic_load_fw
sudo modprobe aic8800_fdrv
```

然后拔出并重新插入网卡。等待数秒，让设备依次完成存储模式弹出、固件下载和 Wi-Fi 接口创建。

### 5. 验证

```bash
lsmod | grep -E '^aic|^cfg80211'
nmcli device status
nmcli device wifi list
journalctl -k -b | grep -i aic
```

成功时应能看到：

- `aic_load_fw` 和 `aic8800_fdrv` 已加载
- 类似 `wlx40a5ef5f040a` 的 Wi-Fi 接口
- `nmcli device wifi list` 能列出附近热点
- 内核日志包含 `fw download complete` 和 `New interface create wlan0`

### 6. 卸载

```bash
sudo modprobe -r aic8800_fdrv aic_load_fw
cd aic8800_linux_drvier/drivers/aic8800
sudo make uninstall
sudo rm -rf /lib/firmware/aic8800D80
sudo rm -f /etc/udev/rules.d/aic.rules
sudo udevadm control --reload-rules
```

### 内核升级说明

本驱动是内核外置模块。升级内核后，需要为新内核安装对应头文件，并重新执行编译、`sudo make install` 和模块加载步骤。

### 故障排查

- 看不到 USB 设备：执行 `lsusb`，并重新插入网卡。
- 设备停在 `a69c:5721`：确认 `eject` 已安装、udev 规则存在，然后重新插入网卡。
- 设备停在 `a69c:8d80`：确认 `aic_load_fw` 已加载，且固件位于 `/lib/firmware/aic8800D80/`。
- 有接口但扫描为空：重启无线管理服务：

  ```bash
  sudo systemctl restart wpa_supplicant.service NetworkManager.service
  ```

- 模块格式错误：确认驱动是针对当前 `uname -r` 编译的，并检查 `modinfo` 中的 `vermagic`。

详细的 Linux 6.8 修改记录见 [docs/PORTING_REPORT.md](docs/PORTING_REPORT.md)。

## English

### Verified functionality

- Firmware download from boot-stage device `a69c:8d80`, followed by re-enumeration as `a69c:8d83`
- 2.4 GHz and 5 GHz Wi-Fi scanning
- WPA/WPA2 association
- IPv4 configuration through DHCP
- NetworkManager integration
- Module reload and physical USB reconnect

### Supported environment

The current version was tested with:

- Ubuntu 22.04
- Linux `6.8.0-124-generic`
- x86_64
- AIC8800D80 USB Wi-Fi with VID:PID `a69c:5721`, `a69c:8d80`, or `a69c:8d83`

Compatibility branches for older kernels remain in the source, but other distributions and kernel versions have not been tested individually.

### 1. Install build dependencies

```bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r) eject
```

Check that the kernel build directory exists:

```bash
test -e /lib/modules/$(uname -r)/build && echo "kernel headers ready"
```

### 2. Install firmware and the udev rule

Run these commands from the repository root:

```bash
sudo install -d /lib/firmware/aic8800D80
sudo install -m 0644 aic8800_linux_drvier/fw/aic8800D80/* /lib/firmware/aic8800D80/
sudo install -m 0644 aic8800_linux_drvier/tools/aic.rules /etc/udev/rules.d/aic.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

The udev rule ejects the adapter's read-only storage device so it can switch from `a69c:5721` to Wi-Fi boot mode.

### 3. Build the driver

```bash
cd aic8800_linux_drvier/drivers/aic8800
make clean
make
```

A successful build creates:

```text
aic_load_fw/aic_load_fw.ko
aic8800_fdrv/aic8800_fdrv.ko
```

The vendor source emits several non-fatal `no previous prototype` warnings. The build itself should finish with exit status 0.

### 4. Install and load the modules

From `aic8800_linux_drvier/drivers/aic8800`, run:

```bash
sudo make install
sudo modprobe cfg80211
sudo modprobe aic_load_fw
sudo modprobe aic8800_fdrv
```

If an older build is already loaded, replace it with the installed modules:

```bash
sudo modprobe -r aic8800_fdrv aic_load_fw
sudo modprobe aic_load_fw
sudo modprobe aic8800_fdrv
```

Physically reconnect the adapter. Allow a few seconds for storage-mode ejection, firmware download, and Wi-Fi interface creation.

### 5. Verify the installation

```bash
lsmod | grep -E '^aic|^cfg80211'
nmcli device status
nmcli device wifi list
journalctl -k -b | grep -i aic
```

A working installation shows:

- Loaded `aic_load_fw` and `aic8800_fdrv` modules
- A Wi-Fi interface similar to `wlx40a5ef5f040a`
- Nearby access points in `nmcli device wifi list`
- `fw download complete` and `New interface create wlan0` in the kernel log

### 6. Uninstall

```bash
sudo modprobe -r aic8800_fdrv aic_load_fw
cd aic8800_linux_drvier/drivers/aic8800
sudo make uninstall
sudo rm -rf /lib/firmware/aic8800D80
sudo rm -f /etc/udev/rules.d/aic.rules
sudo udevadm control --reload-rules
```

### Kernel upgrades

This is an out-of-tree kernel module. After a kernel upgrade, install the matching kernel headers and repeat the build, `sudo make install`, and module-loading steps for the new kernel.

### Troubleshooting

- USB device is absent: run `lsusb` and physically reconnect the adapter.
- Device remains at `a69c:5721`: verify that `eject` is installed and the udev rule exists, then reconnect the adapter.
- Device remains at `a69c:8d80`: verify that `aic_load_fw` is loaded and that firmware files exist under `/lib/firmware/aic8800D80/`.
- Interface exists but scans return no results: restart the wireless management services:

  ```bash
  sudo systemctl restart wpa_supplicant.service NetworkManager.service
  ```

- Invalid module format: make sure the module was built for the current `uname -r` and inspect its `vermagic` with `modinfo`.

See [docs/PORTING_REPORT.md](docs/PORTING_REPORT.md) for the Linux 6.8 porting and validation record.

