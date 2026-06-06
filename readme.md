Ubuntu 6.8 Wi-Fi 驱动安装——针对 MT7902 无线网卡 

适用于：

```text
Ubuntu 22.04
Kernel 6.8.0-117-generic
MediaTek MT7902 PCIe Wi-Fi
PCI ID: 14c3:7902
驱动模块: mt7902e
```

## 结果

已验证可用：

```text
Wi-Fi 扫描：正常
Wi-Fi 连接：正常
DHCP：正常
DNS：正常
联网：正常
```

## 安装步骤

### 1. 安装依赖

```bash
sudo apt update
sudo apt install -y git build-essential linux-headers-$(uname -r) linux-firmware
```

### 2. 克隆驱动

```bash
cd ~
git clone https://github.com/hmtheboy154/mt7902.git
cd mt7902
```

### 3. 修复连接失败 error -22

未修改时可能出现：

```text
failed to insert STA entry for the AP (error -22)
```

执行补丁：

```bash
python3 <<'PY'
from pathlib import Path
import re

target = None

for p in Path(".").rglob("mac80211.c"):
    s = p.read_text(errors="ignore")
    if "mt76_vif_phy" in s and "mlink->ctx" in s:
        target = p
        break

if target is None:
    raise SystemExit("Cannot find mac80211.c")

s = target.read_text(errors="ignore")
old = s

s = re.sub(
    r'if\s*\(\s*!mlink->ctx\s*\)\s*\n\s*return\s+NULL\s*;',
    'if (!mlink->ctx)\n\t\treturn hw->priv;',
    s
)

s = re.sub(
    r'if\s*\(\s*!mlink->ctx\s*\)\s*return\s+NULL\s*;',
    'if (!mlink->ctx)\n\t\treturn hw->priv;',
    s
)

target.write_text(s)
print("patched file:", target)
print("changed:", old != s)
PY
```

### 4. 编译安装

```bash
make clean 2>/dev/null || true
make -j$(nproc)

sudo make install -j$(nproc)
sudo make install_fw
sudo depmod -a
sudo update-initramfs -u -k all
sudo reboot
```

## 验证

重启后执行：

```bash
lsmod | grep -E "mt7902e|mt7902|mt76"
lspci -nnk | grep -A5 -i "7902\|mediatek\|network"
nmcli device status
```

正常结果应包含：

```text
Kernel driver in use: mt7902e
wlp8s0    wifi    已连接
```

联网测试：

```bash
ip addr show wlp8s0
ip route
ping -c 4 223.5.5.5
ping -c 4 baidu.com
```

## 内核升级后重装

内核升级后可能需要重新编译：

```bash
cd ~/mt7902
make clean
make -j$(nproc)

sudo make install -j$(nproc)
sudo make install_fw
sudo depmod -a
sudo update-initramfs -u -k all
sudo reboot
```


本记录只解决 Wi-Fi，蓝牙未包含。
