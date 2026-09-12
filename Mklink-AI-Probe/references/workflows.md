# 工作流程与错误处理

> 触发词：首次烧录、RTT 集成、故障排查
> 返回索引：[SKILL.md](../SKILL.md)

## 工作流程

### 新项目首次烧录

1. 运行 `python -m mklink project-init` 解析工程与目标配置。
2. 工程 IDE 可用时，优先通过 IDE 原生命令行编译并下载。Keil 默认依次执行 `UV4.exe -b` 和 `UV4.exe -f`。
3. IDE 不可用、不适用或只有预编译镜像时，使用 pyOCD 在线烧录。
4. 前两种能力都不可用或用户要求脱机部署时，使用 MKLink 脱机下载 API。

完整的停止条件、命令和 FLM 来源顺序见 [firmware-download-priority.md](firmware-download-priority.md)。`python -m mklink flash` 仅在用户明确要求原生串口/FLM 兼容路径时使用。

### 编译、下载 + 查看 RTT

1. 按 [firmware-download-priority.md](firmware-download-priority.md) 选择后端；IDE 工程默认先编译再下载。
2. 检查编译/下载日志，并完成 Flash 回读或后端 verify。
3. 执行 `python -m mklink rtt --duration 15`，确认新固件运行输出。

某个后端一旦开始执行后失败，必须停止并报告根因，不能静默切换到下一后端。

### RTT 首次集成

```bash
# 1. 集成 RTT 源码到项目（自动检测工程类型和头文件路径）
python -m mklink rtt-integrate --project-root .

# 2. 在 Keil/IAR 中重新编译项目（手动）

# 3. 按固件下载优先级完成下载，再查看 RTT
python -m mklink rtt
```

**生产固件：** 从工程定义中移除 `USE_RTT` 宏即可禁用所有 RTT 输出。

### VPN/局域网现场机直连

1. 先读 [commands-remote.md](commands-remote.md)，确认现场机只部署官方独立
   Site Agent，工程师机才读取本 Skill。
2. 现场维护者先在回环地址完成 readiness/health 验证；需要监听受管 VPN/局域网
   地址时，显式使用 `--allow-lan` 并配置 token。
3. 工程师机从环境变量读取 token，用 `sites add` 注册
   `ws://<VPN_OR_LAN_HOST>:<PORT>`，再用 `sites use` 设置项目 active pointer。
4. 按 `health` → `status` → `capabilities` → `ports` 顺序检查监听器、协议协商、
   能力和探针状态；不要把网络故障误当成探针故障。
5. 普通读操作可通过 SDK、CLI 或可选 MCP 执行。文件先原子上传并取得 opaque
   reference；只有后续消费该 reference 的操作才会产生设备影响。
6. 烧录、擦除、写内存/变量、上传后激活、停止或替换现场 Agent 前，向用户展示
   站点、目标、输入摘要和影响，取得本地明确授权，再使用 CLI `--yes` 或 MCP
   `confirm=True`。替换 Agent 文件只能由现场维护者在旧进程退出后完成。

---

## 错误处理

### Web GUI 导入 Pack 后仍找不到 SVD

1. 先区分“Pack/SVD 可解析”和“当前运行实例已刷新”。读取
   `%LOCALAPPDATA%\MKLink\pyocd\state.json`，确认 Pack 已登记且归档仍存在；再请求
   `/api/health` 取得实际 `backend_port`，并请求
   `/api/dash/superwatch/peripherals/targets?q=<器件前缀>`。离线解析成功不能证明运行
   实例的目录已经更新。
2. Pack 已登记、离线发现有目标，但运行接口仍返回 0 个目标时，优先怀疑旧的内存
   目录或孤立的 `python -m mklink serve`。关闭/重开浏览器页面不等于 Python 后端
   已退出；检查进程创建时间和父进程，不要反复导入同一 Pack。
3. 只按监听端口定位后端并核对命令行，禁止按进程名清除所有 `python.exe`：

   ```powershell
   $listener = Get-NetTCPConnection -State Listen -LocalPort <PORT>
   $backend = Get-CimInstance Win32_Process -Filter "ProcessId = $($listener.OwningProcess)"
   $backend | Select-Object ProcessId, ParentProcessId, CreationDate, CommandLine
   ```

   只有命令行确认包含 `-m mklink serve` 且端口与 `/api/health` 一致时，才处理该
   PID。服务仍响应时先停止 SuperWatch/RTT/SystemView 等活动流并调用设备断开；保存
   原启动参数后再终止精确 PID，确认端口已释放，然后由原桌面入口或原命令重新启动。
4. 新实例启动后再次检查 `/api/health` 和 targets 接口。设备连接响应若含
   `target_initializing: true`，等待 `/api/device/status` 进入 `READY` 后再选 SVD；
   初始化期间的选择可能被后续初始化覆盖。状态为 `DUMP_STREAM` 或 SuperWatch 为
   `running` 时，必须先调用 `/api/dash/superwatch/stop`，否则不能切换外设芯片。
5. 使用 PDSC 中的精确器件名搜索，例如 `APM32E103RC`，不要把完整订货号直接当作
   Pack 器件名。选择后核对返回的 `selection.pack`、`selection.svd` 和非零
   `items`。SVD 无需 AXF，但开始读取寄存器前，所选 SVD 必须与实际目标芯片一致。

本次验证基线：`Geehy.APM32E1xx_DFP@1.0.4` 对 `APM32E1` 返回 8 个型号，使用
`SVD/APM32E103xx.svd`，可生成 6077 个外设寄存器/字段项。若这些离线事实成立而运行
接口为空，应处理运行实例和缓存，不应修改厂商 Pack。

| 场景 | 处理方式 |
|------|----------|
| Pack 已导入但 SVD 型号列表为空 | 按上面的 Web GUI/SVD 流程核对实际端口、孤立 Python 后端、目录缓存和活动 SuperWatch 流 |
| COM 口不存在 | `python -m mklink discover` 查找端口 |
| IDCODE 无效 | 检查 SWD 接线和目标板供电 |
| 新 MCU 未知 / profile 缺失 | 先按内置 Pack、内置 DAPLink FLM、已安装 Pack、自定义 FLM 顺序解析；仍无匹配时运行 `python -m mklink mcu-detect`，多候选再选择内部 Flash FLM 固化 |
| 找不到 H723/H7 等 FLM | 先确认发布包内置 Pack/DAPLink 算法；再检查已安装 Keil/Arm Pack；最后才使用显式自定义 `--flm` |
| FLM 加载失败 | 先 `python -m mklink mcu-detect` 确认 profile/FLM，再 `python -m mklink copy-flm` 拷贝 FLM |
| RTT 搜索失败 | 检查固件是否已集成 RTT 并重新编译 |
| RTT 集成验证失败 | 确认 `main()` 在合适位置调用了 `SEGGER_RTT_Init()`（通常在系统初始化之后） |
| 头文件目录不存在 | 检查项目的 Include Path 配置，使用 --inc-dir 指定正确路径 |
| HEX 文件未找到 | 先编译项目，再运行 `python -m mklink project-init` 更新路径 |
| 项目未配置 | `python -m mklink project-init` |
| 远程 listener 不可达 | 先检查受管 VPN/局域网可达性、现场 Agent 进程和 host/port；不要打印 token，也不要把 `remote reconnect` 当成网络修复 |
| 远程认证失败 | 确认工程师侧 `--token-env` 与现场 Agent 的环境变量或 owner-only token file 指向同一密钥；不要把密钥复制到 URL、日志或聊天 |
| `health` 成功但探针未连接 | 用 `status`/`ports` 检查现场探针，再运行 `remote reconnect`；该操作只重连探针 |
| capability unavailable | 以 `remote capabilities` 的协商结果为准；停止调用未发布能力，不猜测 operation 或参数 |
| 上传中断或校验失败 | 重新执行 `remote upload`；协议会中止未完成 session，不支持续传，也不接受客户端指定现场路径 |
| 高风险操作被拒绝 | 回到工程师本地确认站点、目标和影响；CLI 添加 `--yes`，MCP 传 `confirm=True`，SDK 传 `confirm=True`，不得绕过现场 Agent 的二次校验 |
