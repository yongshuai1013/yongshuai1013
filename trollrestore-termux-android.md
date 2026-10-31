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

採用 [usedoperative-sudo/termux-usbmuxd](https://github.com/usedoperative-sudo/termux-usbmuxd) 方案（以 Termux:API 的 USB 存取餵給 usbmuxd）。

> **注意：termux-api 本體是一個 APK 檔**——即 Termux:API App，跟 Termux 本體一樣從 F-Droid 以 APK 安裝在手機上，`termux-usb` 靠它存取 USB。它是事先裝好的 App，不走 pkg，所以不在上面的 pkg 安裝清單裡。

```bash
pkg install usbmuxd libimobiledevice
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
...[truncated 2324 chars]個原因都不能用：

1. 它的依賴含 qh3、av 等要在 Android 上現編的套件，裝不上；
2. **7.0 起移除了 `cli.cli_common.Command`**，而 trollstore.py 直接 import 它——最新版在任何平台都跑不起來 TrollRestore。

改用釘版＋`--no-deps`，只裝實際 import 鏈需要的純 Python 套件（`construct-typing` 必須釘 0.7.0，新版驗證變嚴會讓 6.2.0 在 import 時就爆）：

```bash
pip install --no-deps --no-build-isolation pymobiledevice3==6.2.0 construct-typing==0.7.0 bpylist2 rich requests click urllib3 idna certifi packaging coloredlogs humanfriendly hexdump inquirer3 blessed jinxed wcwidth readchar editor pygments construct ifaddr ipython traitlets nest-asyncio hyperframe tqdm parameter-decorators typing_extensions pygnuutils xonsh xmod
```

> requests 會警告缺 chardet／charset_normalizer，本流程用不到，可忽略。

**pip 全滅時的診斷（本次實際踩到）**：若 pip 裝任何東西都在 install 階段 ImportError，先跑這支診斷指令逼出真正的錯誤：

```bash
python3 -c "import pip._internal.operations.install.wheel"
```

本次最底層錯誤是 pyexpat 載入失敗（`cannot locate symbol "XML_SetHashSalt16Bytes"`），根因是 Termux 的 libexpat 與 pyexpat 版本不對版；pip 26.x 的 audit hook 會把這個失敗放大成「任何安裝都過不了 install 階段」。修法是 `pkg reinstall libexpat`（pip 本體無辜，不用重裝 pip）。

## 步驟三：pymobiledevice3 的三處 Android 補丁

套件路徑以下以 `$PY=$PREFIX/lib/python3.14/site-packages/pymobiledevice3` 表示。

**補丁 1：讓它認得 Android 平台**——`osu/os_utils.py` 以 `sys.platform` 分派系統，只認 linux／win32／darwin，Termux 回報 `android` 直接拋 `OSNotSupportedError`：

```bash
sed -i 's/elif cls._os_name == "linux":/elif cls._os_name in ("linux", "android"):/' $PY/osu/os_utils.py
```

**補丁 2：/proc 讀不到時不要死**——同檔的 `is_wsl()` 讀 `/proc/version`，Android 不讓 App UID 讀、拋 PermissionError，而原碼只接 FileNotFoundError：

```bash
sed -i 's/except FileNotFoundError:/except (FileNotFoundError, PermissionError):/' $PY/osu/os_utils.py
```

**補丁 3：改掉寫死的 usbmuxd socket 預設路徑**——`usbmux.py` 的 `USBMUXD_PIPE` 寫死 `/var/run/usbmuxd`；TrollRestore 內部 sparserestore 的第二次連線不吃 `--usbmux` 參數、只走這個預設值，在 Termux 上必然 FileNotFoundError：

```bash
sed -i 's|USBMUXD_PIPE = "/var/run/usbmuxd"|USBMUXD_PIPE = "/data/data/com.termux/files/usr/var/run/usbmuxd"|' $PY/usbmux.py
```

## 步驟四：執行 TrollRestore

前置：**先在 iPhone 上關閉「尋找我的 iPhone」**（裝完再打開）；iPhone 保持解鎖、USB 線不要拔；usbmuxd 已依步驟一起好。

```bash
cd ~/TrollRestore
python3 trollstore.py --usbmux $PREFIX/var/run/usbmuxd
```

- 它不讀 `USBMUXD_SOCKET_ADDRESS` 環境變數，socket 路徑靠 `--usbmux` 參數給（該參數是 TrollRestore 內建的）。
- 出現 `Enter the app name:` 時輸入要覆蓋的系統 App 名稱：**Tips**（找不到再試中文顯示名「提示」）。
- 成功會印出 `Replacing Tips.app with TrollStore Helper.` 接著 `Rebooting device`，iPhone 自動重開機，腳本正常結束。

## 步驟五：iPhone 上收尾

1. 打開 **Tips**（已變成 TrollStore Helper）→ 點 **Install TrollStore**。
2. 打開 TrollStore → Settings → **Install Persistence Helper** → 選 **Tips**（同一個 App）。
3. 把 **「尋找我的 iPhone」重新打開**。
4. TrollStore → **Refresh App Registrations**，先前由 TrollStore 安裝的 App 即全部恢復可用。

## 踩坑速查表

| 症狀 | 根因 | 解法 |
|---|---|---|
| pip 裝 cryptography／Pillow／psutil 失敗 | Android 無 wheel、sdist 現編必敗 | 改用 `pkg install python-cryptography python-pillow python-psutil` |
| pip 裝什麼都在 install 階段 ImportError | libexpat 與 pyexpat 版本不對版，被 pip 26.x audit hook 放大 | 診斷：`python3 -c "import pip._internal.operations.install.wheel"`；修法：`pkg reinstall libexpat` |
| `No module named '_cffi_backend'` | Termux 沒把 cffi 打進任何 pkg 套件 | `pkg install clang pkg-config` 後 `pip install --no-build-isolation cffi` |
| `OSNotSupportedError` | `sys.platform == "android"` 不在支援清單 | 補丁 1 |
| `PermissionError: '/proc/version'` | Android 擋 App 讀 /proc | 補丁 2 |
| perform_restore 時 `FileNotFoundError` 連 socket | 內部連線走寫死的 `/var/run/usbmuxd` | 補丁 3 ＋ 執行時帶 `--usbmux` |
| TrollInstallerX 卡在 Exploiting kernel (landa) 閃退 | 15.8.7 起該路已被官方指南劃掉 | 改走本流程（TrollRestore） |
