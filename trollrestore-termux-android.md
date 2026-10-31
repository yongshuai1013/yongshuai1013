# 用 Android（Termux）跑 TrollRestore 救回 TrollStore

> 沒有電腦時，用一台 Android 手機當「電腦」，透過 USB 對 iPhone 跑 TrollRestore，把被系統更新洗掉的 TrollStore 裝回來。
> **2026-10-31 實測成功**：iPhone 7（iOS 15.8.8）全程跑完，TrollStore 與其安裝的 App 全部恢復。

## 適用情境

- iPhone 已更新到 **iOS 15.8.7 / 15.8.8**，TrollStore 本體與它裝的 App 全部打不開，Persistence Helper 宿主也被更新洗掉。
- 依 ios.cfw.guide 現行支援表，這兩個版本重裝 TrollStore **只剩 TrollRestore 一條路**：TrollInstallerX 的 landa exploit 在 15.8.8 會卡在第四步「Exploiting kernel (landa)」後閃退（實測多次、原包名與改包名版皆然）。
- 手邊沒有電腦，但有一台 Android 手機（本文用已 root 的小米 5；本文的 usbmuxd 方案本身免 root 也可）。

## 實測環境

| 項目 | 版本 |
|---|---|
| iPhone | iPhone 7（iPhone9,3），iOS 15.8.8 |
| Android | 小米 5（已 root），Termux |
| Termux Python | 3.14（套件路徑 `$PREFIX/lib/python3.14/site-packages/`） |
| pymobiledevice3 | **6.2.0**（必須釘版，原因見下） |
| TrollRestore | JJTech0130/TrollRestore，main 分支 |

## 步驟一：USB 通道（Termux ↔ iPhone）

採用 [usedoperative-sudo/termux-usbmuxd](https://github.com/usedoperative-sudo/termux-usbmuxd) 方案（以 Termux:API 的 USB 存取餵給 usbmuxd）：

```bash
pkg install usbmuxd libimobiledevice termux-api
# 依該專案說明把 termux-usbmuxd wrapper 放進 $PREFIX/bin/
```

每次使用時（接上 USB 線後）：

```bash
termux-usb -l                                   # 列出 USB 裝置，記下路徑（例如 /dev/bus/usb/001/002）
termux-usbmuxd /dev/bus/usb/001/002             # 啟動 usbmuxd
export USBMUXD_SOCKET_ADDRESS=UNIX:$PREFIX/var/run/usbmuxd
idevicepair pair                                # 首次配對（iPhone 上點「信任」）
ideviceinfo                                     # 能印出裝置資訊即連通
```

## 步驟二：裝 Python 依賴（本流程最容易卡關的一段）

先 clone 本體：

```bash
pkg install python git clang pkg-config
git clone https://github.com/JJTech0130/TrollRestore
```

### 2-1. 編譯型套件改用 Termux 預編版

cryptography、Pillow、psutil 在 Android 上沒有 pip wheel，pip 抓 sdist 現編（Rust/C）必敗，改用 pkg 預編版頂掉：

```bash
pkg install python-cryptography python-pillow python-psutil
```

### 2-2. 補裝 cffi

Termux 的 python-cryptography 執行時仍需要 `_cffi_backend`，但它不在任何 pkg 套件裡，要用 pip 現編（需要 clang）：

```bash
pip install --no-build-isolation cffi
python3 -c "import _cffi_backend; print('ok')"
```

### 2-3. 釘版安裝 pymobiledevice3（不要直接裝 requirements.txt）

直接 `pip install -r requirements.txt` 會拉最新版 pymobiledevice3，兩
...[truncated 2324 chars]