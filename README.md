# vanet_final_project

# 從 0 到成功跑起 Veins + SUMO + OMNeT++ 6.0.3

**（Ubuntu 22.04 / VMware / SUMO 1.21 / Veins 5.3.1 測試）**
以下逐步紀錄 **完整可重製流程、常見死角與排錯方法**。
只要嚴格照做，任何新手都能在約 30 – 40 分鐘內跑出第一條 DSRC 封包。

---

## 0 — 系統前置

| 元件                            | 版本                       | 來源                                                                          |
| ----------------------------- | ------------------------ | --------------------------------------------------------------------------- |
| Ubuntu                        | 22.04 LTS（桌面或 Server 都可） | 官方 ISO                                                                      |
| g++ / make / bison / qttools5 | apt 預設倉                  | `sudo apt install build-essential qtbase5-dev bison libopenmpi-dev` 等       |
| Python                        | 3.10+                    | OS 內建                                                                       |
| **SUMO**                      | 1.21.0                   | [https://sumo.dlr.de/releases/](https://sumo.dlr.de/releases/) 解壓到 `~/sumo` |

```bash
# 下載並解壓
cd ~
wget https://sumo.dlr.de/releases/1.21.0/sumo-src-1.21.0.tar.gz
tar -xzf sumo-src-1.21.0.tar.gz && mv sumo-1.21.0 sumo
```

---

## 1 — 一鍵滿足 Python 依賴

Ubuntu 近版用 PEP 668 限制 `pip`；加 `--break-system-packages` 可安全安裝：

```bash
sudo apt update
sudo apt install -y python3-numpy python3-scipy python3-pandas python3-matplotlib
sudo pip3 install --break-system-packages posix_ipc traci sumolib
```

---

## 2 — 展開並編譯 OMNeT++ 6.0.3

```bash
cd ~
wget https://github.com/omnetpp/omnetpp/releases/download/omnetpp-6.0.3/omnetpp-6.0.3-linux-x86_64.tgz
tar -xzf omnetpp-6.0.3-linux-x86_64.tgz
cd omnetpp-6.0.3
echo 'WITH_NETBUILDER=yes' >> configure.user     # 啟用動態 NED
source setenv
./configure
make -j$(nproc)          # 約 10–15 分鐘
```

> 只要結尾看到 **“Now you can type 'omnetpp' to start the IDE.”** 即代表核心 OK，警告可忽略。

---

## 3 — 取得 Veins 5.3.1 並編譯

```bash
cd ~/omnetpp-6.0.3/samples
git clone https://github.com/sommer/veins.git
cd veins && git checkout veins-5.3.1
# 生成並編譯 libveins.so
make -C src -j$(nproc)
```

---

## 4 — 設定環境變數（\~/.bashrc）

```bash
cat >> ~/.bashrc <<'EOF'
# --- SUMO & OMNeT++ ---
export SUMO_HOME=$HOME/sumo
export PATH=$SUMO_HOME/bin:$PATH
export PYTHONPATH=$SUMO_HOME/tools:$PYTHONPATH
EOF
source ~/.bashrc
```

驗證：

```bash
which sumo          # -> ~/sumo/bin/sumo
python3 -c "import traci, sumolib; print('traci OK')"
```

---

## 5 — 範例專案：`examples/veins`

我們選用 **Veins 自帶最精簡的例子**，手動產生一個十字路口：

```bash
cd ~/omnetpp-6.0.3/samples/veins/examples/veins
mkdir -p sumo

# 5.1 產生 2×2 Grid 路網
netgenerate --grid --grid.x-number=2 --grid.y-number=2 --grid.length=100 \
            -o sumo/cross.net.xml

# 5.2 建立最小 SUMO 設定
cat > sumo/cross.sumo.cfg <<'CFG'
<configuration>
  <input>
    <net-file value="cross.net.xml"/>
  </input>
  <time>
    <begin value="0"/>
    <end   value="200"/>
  </time>
</configuration>
CFG
```

---

## 6 — 修改 `omnetpp.ini`

```ini
[Config Default]
# 讓 Veins 用 launchd 自動開 SUMO
*.manager.launchConfigFile = "../../bin/veins_launchd"
*.manager.configFile       = "sumo/cross.sumo.cfg"
*.manager.sumoExecutable   = "sumo"          # 使用命令列版

# 若 GUI 很卡可加：
# *.manager.sumoGUI = false
```

---

## 7 — 第一次執行（文字介面）

```bash
cd ~/omnetpp-6.0.3 && source setenv
cd samples/veins/examples/veins
./run -u Cmdenv -c Default
```

◎ **成功輸出**（精簡）

```
INFO: Launching SUMO: sumo -c sumo/cross.sumo.cfg --remote-port 32955
INFO: TraCIScenarioManager connected to TraCI server
End. (Simulation time limit reached -- at t=200s)
```

---

## 8 — 若遇到常見錯誤對照表

| 錯誤訊息                                            | 原因                                 | 修復                                                        |
| ----------------------------------------------- | ---------------------------------- | --------------------------------------------------------- |
| **`No such file or directory: cross.sumo.cfg`** | 路徑錯                                | `*.manager.configFile` 指向正確檔案                             |
| **`Please declare env var 'SUMO_HOME'`**        | launchd 找不到 SUMO                   | `.bashrc` 加 `export SUMO_HOME`、重新 `source`                |
| **`ImportError: traci`**                        | Python 模組缺                         | `sudo pip3 install --break-system-packages traci sumolib` |
| **`Connection refused`**                        | SUMO 未啟動/立刻崩潰                      | 手動 `sumo -c ... --remote-port 9999` 驗證，或查看 launchd stderr |
| **`env: ‘opp_run’: No such file`**              | 忘了 `source ~/omnetpp-6.0.3/setenv` | 每開新終端機都要載                                                 |

---

## 9 — 顯示 GUI（可選）

```bash
# 先手動開 SUMO GUI
sumo-gui -c sumo/cross.sumo.cfg --remote-port 9999 --start

# 再開 Qtenv
./run -u Qtenv -c Default
```

> 在 VM 裡 GUI 若黑畫面，可按 `r` (reset view) 或改用命令列 SUMO。

---

## 10 — 開始開發安全 DSRC

1. **SecureApp 建置**

   ```
   cp src/veins/modules/application/ieee80211p/DemoBaseApplLayer.{cc,h} \
      src/veins/modules/application/ieee80211p/SecureApp.{cc,h}
   ```
2. **SecureHelper**
   *放在* `src/veins/base/security/`

   * `generateKey()`, `signECDSA()`, `verifyECDSA()`, `encryptAES()`, `decryptAES()`
   * 使用 OpenSSL 或 Crypto++。
3. **連結到節點**
   `omnetpp.ini`

   ```ini
   *.node[*].appl.typename = "SecureApp"
   *.useEncryption = true
   ```
4. **重新編譯**

   ```bash
   make -C src -j$(nproc)
   ```
5. **不同密度測試**

   * `netgenerate --grid ...` 變更長寬 / 匯入 OSM
   * `randomTrips.py` 產生流量
   * `omnetpp.ini` 用 `repeat=...` 批次跑

---

## 11 — 完整排錯流程（捷徑）

1. **確定 OMNeT++ & SUMO 路徑**
   `source setenv && which sumo && python3 -c "import traci"`
2. **手動跑 SUMO**
   `sumo -c <cfg> --remote-port 9999` 必須進入「Waiting for a connection」。
3. **Veins 只連線**

   ```
   *.manager.moduleType      = "veins::TraCIScenarioManager"
   *.manager.launchConfigFile = ""
   *.manager.port            = 9999
   ```
4. **一次成功後，再開啟 veins\_launchd 全自動**。

---

### 你已完成最困難的「環境能跑」階段。

後續只需專注於 **應用層安全邏輯與效能評估**；任何新的編譯錯誤、TraCI 連線問題或 SUMO 路網疑難，對照本指南或將訊息全貼出，即可迅速定位。祝專案順利!
