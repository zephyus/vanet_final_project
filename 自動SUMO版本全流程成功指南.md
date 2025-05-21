## 一條龍復現指南：從 SUMO 編譯到 Veins 樣例跑通

（Ubuntu 22.04／OMNeT++ 6.0.3／Veins 5 － **繁體中文**）

---

### 0 前置環境

| 元件        | 版本     | 重點                             |
| --------- | ------ | ------------------------------ |
| GCC / g++ | ≥ 11   | C++17 相容                       |
| CMake     | ≥ 3.18 |                                |
| Python    | 3.12   | 用於 SUMO tools                  |
| SUMO      | 1.20.0 | 原始碼自行編譯                        |
| OMNeT++   | 6.0.3  | 已 `./configure && make -j`     |
| Veins     | 5.x    | `samples/veins/examples/veins` |

---

## 1 編譯 SUMO 1.20.0

```bash
git clone https://github.com/eclipse/sumo.git -b v1_20_0
cd sumo
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release        # GUI 旗標無效也沒關係
make -j$(nproc)
```

SUMO 會把可執行檔輸出到 **`$SUMO_SRC/bin`**，
確認：

```bash
export SUMO_HOME="$HOME/sumo"
export PATH="$SUMO_HOME/bin:$PATH"
export PYTHONPATH="$SUMO_HOME/tools:$PYTHONPATH"

sumo --version        # Eclipse SUMO 1.20.0
python3 -c "import traci, sumolib; print('Traci OK, Sumolib OK')"
```

---

## 2 Veins 樣例場景準備

進入樣例資料夾並**確保 SUMO 檔案齊全**：

```bash
cd ~/omnetpp-6.0.3/samples/veins/examples/veins

# 若路由檔不存在，利用 randomTrips 生成
python3 $SUMO_HOME/tools/randomTrips.py \
        -n sumo/cross.net.xml \
        -r sumo/cross.rou.xml \
        -e 200                         # 模擬 200 s

# 把三個檔案搬到與 launch.xml 同層
cp sumo/cross.net.xml  .
cp sumo/cross.rou.xml  .
cp sumo/cross.sumo.cfg .
```

---

## 3 launch.xml（給 veins\_launchd）的最小範例

`launch.xml` *必須* 放在 **Veins 範例資料夾根目錄**，且：

* **每個 `<copy>` 的 file 只能是檔名**，不可含 `/`。
* 至少要有一個 `type="config"` 的 `<copy>`。

```xml
<?xml version="1.0"?>
<launch>
    <launcherd host="localhost" port="${port}" numRetries="5" seed="${seed}"/>

    <copy file="cross.net.xml"/>
    <copy file="cross.rou.xml"/>
    <copy file="cross.sumo.cfg" type="config"/>

    <sumo cmd="sumo -c cross.sumo.cfg \
                    --remote-port ${port} \
                    --seed ${seed} \
                    --step-length 0.1 \
                    --time-to-teleport -1 \
                    --quit-on-end=true"/>
</launch>
```

---

## 4 omnetpp.ini 關鍵段落

```ini
[Config Default]
*.manager.moduleType = "veins::TraCIScenarioManagerLaunchd"
*.manager.launchConfig = xmldoc("launch.xml")
*.manager.port = 9999
*.manager.debug = true
*.manager.autoShutdown = true
*.manager.configFile = "cross.sumo.cfg"   # 可有可無，留供 Veins 讀取場景時限
sim-time-limit = 200s
```

> 若要改用 **Forker**（少一個 daemon），把 moduleType 換成
> `veins::TraCIScenarioManagerForker` 並刪掉 `launchConfig` 即可。

---

## 5 執行順序

### 5.1 開守護程式（終端機 A）

```bash
cd ~/omnetpp-6.0.3/samples/veins
bin/veins_launchd -vv -c $SUMO_HOME/bin/sumo -p 9999
# 觀察須出現：
# Copying cross.net.xml / cross.rou.xml / cross.sumo.cfg
# Starting SUMO ...
# Proxying TraCI on port 9999
```

### 5.2 跑 OMNeT++（終端機 B）

```bash
cd ~/omnetpp-6.0.3
source setenv
cd samples/veins/examples/veins
./run -u Cmdenv -c Default
# 成功時：
# INFO: Connected to TraCI server on localhost:9999
# 事件推進直到 t=200s 結束
```

---

## 6 常見錯誤與對策速查表

| 現象                                        | 日誌關鍵字                                        | 解法                                                          |
| ----------------------------------------- | -------------------------------------------- | ----------------------------------------------------------- |
| **CMake 無視 `-DENABLE_GUI=OFF`**           | `Manually-specified variables were not used` | 乾脆不關 GUI；或 `-DFOX_CONFIG=` 再重編                              |
| **Veins 啟動時要求輸入 launchConfig**            | `prompt for user input`                      | 在 `.ini` 指定 `*.manager.launchConfig = xmldoc("launch.xml")` |
| **launchd abort: file name contains “/”** | `contains a "/"`                             | `<copy>` 只能寫檔名；不要含路徑或使用 `<chdir>`                           |
| **launchd abort: file … does not exist**  | `does not exist`                             | 把檔案複製（或 symlink）到與 `launch.xml` 同層                          |
| **OMNeT++ Connection refused**            | `could not connect` (111)                    | 守護程式沒在跑、port 不一致或 launchd 已 abort                           |
| **Connection reset by peer**              | `(104)`                                      | SUMO 被 launchd 提前結束──通常上面兩個原因所致                             |

---

## 7 關於 GUI 與路徑管理

* 禁用 GUI 雖可省依賴，但若旗標無效，直接裝 `libfox-1.6-dev` 亦可。
* production 或 CI 若要無 GUI，可用 `-DFOX_CONFIG=` 重編 SUMO 或選擇 **libsumo**。

---

### ✅ 走完以上步驟，你應該會看到：

```
Running simulation...
** Event #202   t=200   ... 100% completed
End.
```

代表 **SUMO ↔ TraCI ↔ Veins** 全流程正常。
以後只要把 *net/rou/cfg* 三檔與 `launch.xml` 放同層、兩個終端機依序啟動，就可穩定復現。祝研究順利 🚗💨
