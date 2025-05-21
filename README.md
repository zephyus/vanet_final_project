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


# End‑to‑End Guide: Ubuntu 24.04 + SUMO 1.20.0 + OMNeT++ 6.0.3 + Veins (RSUExampleScenario)

> **Goal**  Run the Veins `RSUExampleScenario` headlessly via **TraCI** on a fresh Ubuntu 24.04 LTS VM.
> When you finish this guide, `./run -u Cmdenv -c Default` will reach *t = 200 s* without errors.

---

## 0 .  Tested Software & Versions

| Component | Version                                  | Notes                                          |
| --------- | ---------------------------------------- | ---------------------------------------------- |
| OS        | Ubuntu 24.04.2 LTS                       | Kernel 6.11                                    |
| GCC/G++   | 13.3.0                                   | from distro                                    |
| CMake     | 3.28.3                                   | from distro                                    |
| Python    | 3.12.3                                   | system default                                 |
| SUMO      | 1.20.0                                   | built from source, **GUI compiled but unused** |
| OMNeT++   | 6.0.3                                    | Academic, built from source                    |
| Veins     | shipped in *omnetpp‑6.0.3/samples/veins* |                                                |

---

## 1 .  System Dependencies

```bash
sudo apt update && sudo apt install -y \
    build-essential g++ gcc make cmake git pkg-config \
    libproj-dev libgdal-dev libxerces-c-dev libgl2ps-dev \
    libfox-1.6-dev    # optional; avoids GUI‑build errors
```

> *`libfox-1.6-dev`* is optional but removes headaches—`cmake` tends to compile the GUI anyway.

---

## 2 .  Build & Set up SUMO 1.20.0

```bash
# fetch & build
cd ~ && git clone --branch v1_20_0 https://github.com/eclipse/sumo.git
mkdir sumo/build && cd sumo/build
cmake .. -DCMAKE_BUILD_TYPE=Release   # ENABLE_GUI flags are ignored, accept default
make -j$(nproc)

# env vars (append to ~/.bashrc)
echo 'export SUMO_HOME="$HOME/sumo"'           >> ~/.bashrc
echo 'export PATH="$HOME/sumo/bin:$PATH"'      >> ~/.bashrc
echo 'export PYTHONPATH="$SUMO_HOME/tools:$PYTHONPATH"' >> ~/.bashrc
source ~/.bashrc
```

**Verify**

```bash
sumo --version           # should show 1.20.0
python3 -c "import traci, sumolib; print('Traci OK, Sumolib OK')"
```

---

## 3 .  Build OMNeT++ 6.0.3  (Qt optional)

```bash
cd ~
wget https://github.com/omnetpp/omnetpp/releases/download/opp_6.0.3/omnetpp-6.0.3-src.tgz
tar xzf omnetpp-6.0.3-src.tgz && cd omnetpp-6.0.3
./configure   # add --disable-qtenv if you want pure CLI
make -j$(nproc)
source setenv          # every shell that runs OMNeT++ needs this
```

---

## 4 .  Prepare the Veins sample

```bash
cd ~/omnetpp-6.0.3/samples/veins/examples/veins

# 4.1 generate a route file (if missing)
python3 $SUMO_HOME/tools/randomTrips.py \
        -n sumo/cross.net.xml \
        -r sumo/cross.rou.xml -e 200

# 4.2 place the three key SUMO files **beside launch.xml**
cp sumo/cross.net.xml  .
cp sumo/cross.rou.xml  .
cp sumo/cross.sumo.cfg .
```

> **Why?** `veins_launchd` only copies files that live in the **same directory** as `launch.xml`; filenames must not contain `/`.

---

## 5 .  Create `launch.xml`

```xml
<?xml version="1.0"?>
<launch>
    <launcherd host="localhost" port="${port}" numRetries="5" seed="${seed}"/>

    <!-- files to copy into the temp dir -->
    <copy file="cross.net.xml"/>
    <copy file="cross.rou.xml"/>
    <copy file="cross.sumo.cfg" type="config"/>

    <!-- SUMO command (no path prefixes!) -->
    <sumo cmd="sumo -c cross.sumo.cfg \
                    --remote-port ${port} \
                    --seed ${seed} \
                    --step-length 0.1 \
                    --time-to-teleport -1 \
                    --quit-on-end=true"/>
</launch>
```

Key rules

* `<copy>` lines **must not contain `/`**
* Exactly **one** `<copy>` has `type="config"`.

---

## 6 .  Edit `omnetpp.ini`

```ini
[Config Default]
*.manager.launchConfig = xmldoc("launch.xml")
*.manager.port         = 9999      # must match daemon
*.manager.seed         = 0
*.manager.debug        = true
*.manager.autoShutdown = true
*.manager.configFile   = "sumo/cross.sumo.cfg"  # still useful for timing
network = RSUExampleScenario
```

> Leave `debug-on-errors=true` during first run; set it to `false` later for batch runs.

---

## 7 .  Run the Simulation

### 7.1 start the launch daemon

```bash
cd ~/omnetpp-6.0.3/samples/veins
bin/veins_launchd -vv -c $SUMO_HOME/bin/sumo -p 9999
```

Expected log extract:

```
Copying cross.net.xml
Copying cross.rou.xml
Copying cross.sumo.cfg
Starting SUMO …
Proxying TraCI on port 9999
```

### 7.2 run OMNeT++

```bash
cd ~/omnetpp-6.0.3
source setenv
cd samples/veins/examples/veins
./run -u Cmdenv -c Default
```

成功輸出：

```
INFO: Connected to TraCI server on localhost:9999
…
<!> Simulation time limit reached -- at t=200s
```

---

## 8 .  Frequent Pitfalls & Fixes

| Symptom                                                                                | Cause                                         | Fix                                                                                 |
| -------------------------------------------------------------------------------------- | --------------------------------------------- | ----------------------------------------------------------------------------------- |
| `CMake Warning: Manually-specified variables were not used by the project: ENABLE_GUI` | SUMO CMake ignores these variables            | Accept default build (GUI compiled) or pass `-D FOX_CONFIG=` to *skip* Fox entirely |
| `Aborting on error: name of file to copy "sumo/…" contains a "/"`                      | `<copy>` path contains `/`                    | Move file to same folder and reference by bare filename                             |
| `Aborting on error: launch config contained no <copy> node with type="config"`         | Missing `type="config"` on one `<copy>`       | Add it to the `.cfg` line                                                           |
| `Connection refused` at t = 0                                                          | `veins_launchd` aborted before proxying TraCI | Check daemon window; fix the abort reason above                                     |

---

## 9 .  Alternative: skip launchd (Forker)

If you prefer to avoid a second terminal:

```ini
*.manager.moduleType = "veins::TraCIScenarioManagerForker"
*.manager.command    = "sumo"
*.manager.configFile = "sumo/cross.sumo.cfg"
*.manager.port       = 0        # auto‐assign
```

`Forker` launches SUMO via `fork()` internally, but you lose the nice temp‑dir isolation for batch runs.

---

## 10 .  Batch runs & tips

* Disable interactive traps:

  * `debug-on-errors = false`
  * `cmdenv-interactive = false`
* Collect results in `results/` – configure in `omnetpp.ini`.
* Use different seeds by overriding `--seed <n>` when calling `./run`.

---

## 11 .  Quick checklist (copy‑paste for fresh VM)

```bash
# 0. prerequisites (Ubuntu 24.04)
sudo apt update && sudo apt install build-essential cmake git libfox-1.6-dev libproj-dev libgdal-dev libxerces-c-dev libgl2ps-dev python3 python3-dev python3-pip -y

# 1. build SUMO 1.20.0
cd ~ && git clone --branch v1_20_0 https://github.com/eclipse/sumo.git && \
mkdir sumo/build && cd sumo/build && cmake .. -DCMAKE_BUILD_TYPE=Release && make -j$(nproc)

# 2. env vars
echo 'export SUMO_HOME="$HOME/sumo"'           >> ~/.bashrc
echo 'export PATH="$HOME/sumo/bin:$PATH"'      >> ~/.bashrc
echo 'export PYTHONPATH="$SUMO_HOME/tools:$PYTHONPATH"' >> ~/.bashrc
source ~/.bashrc

# 3. build OMNeT++ 6.0.3
cd ~ && wget https://github.com/omnetpp/omnetpp/releases/download/opp_6.0.3/omnetpp-6.0.3-src.tgz && \
tar xzf omnetpp-6.0.3-src.tgz && cd omnetpp-6.0.3 && ./configure && make -j$(nproc)

# 4. prep sample
source setenv
cd samples/veins/examples/veins && \
python3 $SUMO_HOME/tools/randomTrips.py -n sumo/cross.net.xml -r sumo/cross.rou.xml -e 200 && \
cp sumo/cross.* .

# 5. ensure launch.xml & omnetpp.ini match this README, then:
cd ~/omnetpp-6.0.3/samples/veins && bin/veins_launchd -vv -c $SUMO_HOME/bin/sumo -p 9999 &
cd ~/omnetpp-6.0.3 && source setenv && cd samples/veins/examples/veins && ./run -u Cmdenv -c Default
```

Enjoy reproducible Veins + SUMO simulations! 🚗📡

