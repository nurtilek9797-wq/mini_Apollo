# Mini-Apollo 项目设计文档

版本：v0.1　|　状态：进行中　|　作者：nurtilek9797-wq

---

## 1. 项目定位

一个缩微版 Apollo Cyber RT 风格的组件化运行时 + 两个真实业务模块（monitor、gps_driver）。

**目标**

1. 锻炼 C++ 实战能力：现代 C++（C++17）、多线程、RAII、模板、CMake、gtest
2. 走完"从零到完整项目"的闭环：需求 → 架构 → 编码 → 测试 → CI → 文档
3. 对标公司 ROS → Cyber RT 改造的技术栈，提前在沙盒中跑通组件化思想
4. 重写自己维护的 monitor / fixposition_driver 模块（只参考功能行为，不复制任何代码）

**非目标**

- 不依赖 ROS / ROS2
- 不做感知、规划等算法
- 不追求极致性能

---

## 2. 关键决策

| 决策点 | 选择 | 理由 |
|---|---|---|
| 风格基准 | Apollo Cyber RT（Component + DAG + Channel pub/sub） | 公司正在向 Cyber RT 改造 |
| 语言标准 | C++17 | 现代且成熟，车端主流 |
| 构建系统 | CMake ≥ 3.16，FetchContent 管理第三方依赖 | 行业标准 |
| 主开发平台 | Linux（WSL2） | 与车端、服务器同构 |
| 副平台 | macOS（clang） | 晚上在家继续开发；核心层不用 Linux 专有 API |
| 同步方式 | git（GitHub 私有/公开仓）为唯一通道 | Windows ↔ Mac 无缝切换 |
| 代码来源 | 全部手写 | 不复制 Apollo / 公司代码，只参考设计思想 |

---

## 3. 总体架构

```
                launch/dag.yaml
                      │
              ┌───────▼────────┐
              │   mainboard    │  程序入口：读 DAG 配置，实例化并启动 Component
              └───────┬────────┘
        ┌─────────────┼─────────────┐
        │             │             │
 ┌──────▼─────┐ ┌─────▼──────┐ ┌────▼────────┐
 │ GpsDriver  │ │  Monitor   │ │  (预留扩展)  │
 │ Component  │ │ Component  │ │             │
 └──────┬─────┘ └─────▲──────┘ └─────────────┘
        │ gps 消息     │ 心跳/状态
 ┌──────▼─────────────┴──────┐
 │   core::Channel (Pub/Sub) │  Writer<T> / Reader<T> / Dispatcher
 └───────────┬───────────────┘
        ┌────▼────┐
        │ common  │  logger / time / config(yaml)
        └─────────┘
```

---

## 4. 核心概念设计（Cyber RT 简化版）

### 4.1 Component（组件）

- 所有业务模块的基类，对标 Cyber RT 的 Component
- 生命周期：`Init(config)` → 运行（内部起线程 / 注册回调）→ `Shutdown()`
- 内置心跳：运行期间定时向 `/monitor/heartbeat` channel 发布心跳消息
- 由 mainboard 按 DAG yaml 创建（第一阶段用工厂注册表；二期再用 dlopen 动态加载 .so）

### 4.2 Channel 通信（发布/订阅）

- `Writer<T>`：`Write(msg)` 发布消息
- `Reader<T>`：注册回调 `void OnMessage(const T&)`
- `Dispatcher`：单例，维护 topic → 订阅者列表，写时派发
- 线程模型：第一阶段在 writer 线程内同步派发；二期引入异步队列 + 线程池
- 消息类型：先用普通 struct（POD），第 4 阶段引入 protobuf

### 4.3 mainboard（程序入口）

- 读取 `launch/dag.yaml`，按配置实例化 Component，管理启动/退出顺序
- 优雅退出：捕获 SIGINT（Ctrl+C），按序调用各组件 Shutdown()

---

## 5. 业务模块设计

### 5.1 gps_driver（对标 fixposition_driver）

- 数据源抽象 `ISource`：`FileSource`（从文件回放 NMEA，跨平台兜底）/ `SerialSource`（POSIX termios 串口）
- NMEA 解析器：先支持 GGA、RMC 两条语句
- 解析结果发布为 `Gps` 消息到 `/gps/fix` channel

### 5.2 monitor（对标公司 monitor 模块）

- 订阅 `/monitor/heartbeat`，维护"组件状态表"（存活、心跳间隔、最后收到时间）
- 订阅关键 channel 统计消息频率，频率异常时告警
- 定时把系统状态汇总打印/发布

---

## 6. 目录结构

```
mini_Apollo/
├── CMakeLists.txt          # 顶层构建脚本
├── README.md               # 项目说明 + 构建步骤
├── .clang-format           # Google 风格
├── .gitignore
├── .gitattributes          # * text=auto eol=lf
├── common/                 # logger / time / config
├── core/                   # component / channel / scheduler / node
├── messages/               # 消息定义（第 4 阶段引入 protobuf）
├── modules/
│   ├── monitor/
│   └── gps_driver/
├── launch/                 # dag.yaml
├── app/                    # mainboard 入口 main.cc
├── tests/                  # gtest
├── docs/                   # 设计文档（本文件在此）
└── notes/                  # 学习笔记、复盘
```

---

## 7. 技术选型

| 用途 | 选型 | 说明 |
|---|---|---|
| 日志 | 手写 mini-logger | 练 RAII / 单例 / stream；后期可换 spdlog |
| 配置 | yaml-cpp | FetchContent 引入 |
| 测试 | googletest | FetchContent 引入 |
| 消息 | struct → protobuf | 第 4 阶段引入 |
| 格式化 | clang-format（Google 风格） | 编辑器保存时自动格式化 |
| CI | GitHub Actions | 第 6 阶段 |
| 串口 | 手写 POSIX termios 封装 | FileSource 兜底保证 Mac 可跑 |

---

## 8. 里程碑路线图（按每周约 12 小时估算）

| 里程碑 | 内容 | 周次 | 产出 |
|---|---|---|---|
| M0 | 脚手架：仓库、WSL、CMake、gtest、clang-format | 第 1 周 | 能编译出打印版本号的 main，ctest 全绿 |
| M1 | 基础设施：logger、time、yaml 配置加载 | 第 2–3 周 | common 库 + 单测 |
| M2 | Channel 通信：Writer / Reader / Dispatcher | 第 4–5 周 | 两个组件互通消息的 demo + 单测 |
| M3 | Component 基类 + mainboard + DAG 加载 | 第 6–7 周 | yaml 驱动启动的空壳系统 |
| M4 | gps_driver：FileSource/SerialSource + NMEA 解析 + protobuf | 第 8–9 周 | 回放 NMEA 文件发布 Gps 消息 |
| M5 | monitor：心跳监控 + 频率统计 + 告警 | 第 10–11 周 | 杀掉一个组件，monitor 能报警 |
| M6 | 集成收尾：README、CI、复盘对比 Apollo | 第 12 周 | 完整交付 + 复盘笔记 |

每个里程碑的验收三件套：**可运行的 demo + 通过的测试 + 更新的文档**。

---

## 9. M0 详细任务清单

| # | 任务 | 预计 | 验收标准 |
|---|---|---|---|
| 0.0 | 装 WSL2 + Ubuntu + 工具链 | 1h | WSL 里 `g++ --version`、`cmake --version` 有输出 |
| 0.1 | GitHub 建仓，Windows 和 Mac 各 clone | 1h | 两边互推测试文件能互相看到 ✅（已完成） |
| 0.2 | 手写顶层 CMakeLists + 目录骨架（自己敲，不复制） | 3h | `cmake -B build && cmake --build build` 编出 `mini_apollo_main`，打印版本号 |
| 0.3 | FetchContent 引入 googletest + 一个 dummy 测试 | 2h | `ctest` 全绿 |
| 0.4 | `.clang-format`（Google 风格）+ 编辑器保存时格式化 | 1h | 保存文件自动格式化 |
| 0.5 | README（构建步骤）+ Mac 从零 clone 编译跑通 | 2h | Mac clang 编译通过，测试全绿 |
| 0.6 | 周末复盘：小结哪卡了、CMake 哪不懂 | 1h | 笔记进 `notes/` |

---

## 10. 工作方式约定

1. **开工先写 10 分钟伪代码注释**，再翻译成 C++——专治"大脑空白"
2. 卡壳 20 分钟 → 标 `// TODO: 待研究` 继续走，周末集中复盘
3. 每次收工必 `git commit && git push`；每次开工先 `git pull`
4. Kimi 的角色：架构评审、代码评审、卡点答疑；**代码全部自己手写**
