# FreeRTOS 开发与审阅

面向 Codex 的 FreeRTOS Agent Skill，把任务设计、数据交接、异常处理和验证方法整理成可复用的工程工作流。

**A Codex skill for developing and reviewing FreeRTOS firmware.** Covers task ownership, IPC, ISR handoffs, timing, resource budgets, failure paths, and evidence-based verification. Instructions and examples are primarily in Simplified Chinese.

支持原生 FreeRTOS API，以及已有 CMSIS-RTOS 封装的工程；以项目实际内核版本、端口、驱动和构建配置为准。适用于 STM32 等平台，不绑定特定 MCU、BMS 产品或固定任务架构。

## 它能帮助你做什么

| 模式 | 适合的请求 | 交付内容 |
|---|---|---|
| 开发与修复 | 实现功能、修复缺陷、裸机迁移、工程集成 | 行为契约、实际代码变更、集成入口、验证结果及限制 |
| 设计与解释 | 先设计任务、比较通信机制、解释现有链路 | 所有权、接口、时序与资源预算、取舍及验证方案 |
| 只读审阅 | 审查架构、可靠性、外部框架适用性 | 带文件和符号证据的问题、影响、最小建议和验证条件 |

工作流重点关注：

- **任务和资源所有权**：谁操作外设，谁更新状态，哪些数据需要跨任务交接。
- **通信与同步**：Queue、Notification、Event、Mutex、快照和 DMA 缓冲区的语义及生命周期。
- **时序与资源**：周期、优先级、整体等待预算、Tick 单位、栈/堆和队列容量。
- **异常路径**：创建失败、队列满、锁超时、旧数据、故障送达、看门狗和 Flash 更新中断。
- **工程集成**：活动编译分支、构建入口、初始化顺序、生产者/消费者与生成配置的一致性。
- **验证证据**：区分源码检查、主机测试、固件构建、目标板运行和真实系统验收。

例如：给一次驱动调用加 timeout，仍可能留下无限重试；更新快照时间戳，不代表采样成功；保存请求入队，也不代表参数已经落盘。这些都是本 Skill 会沿代码继续追踪的交接点。

## 安装

仓库提供的是指令与参考资料，不包含 FreeRTOS 内核、固件工程、MCP 服务或运行时依赖。安装本 Skill 不需要 Python；手动克隆需要 Git。

### 方法一：让 Codex 安装

在带有 `skill-installer` 的 Codex 中发送：

```text
$skill-installer 请安装这个 GitHub Skill：
https://github.com/Xylonzzz/freertos-development-review/tree/main/skills/freertos-development-review
```

### 方法二：手动安装

以下命令把 Skill 安装到用户级 `~/.agents/skills`。目标目录已存在时，先保留自己的修改，再决定更新方式。

Windows PowerShell：

```powershell
git clone https://github.com/Xylonzzz/freertos-development-review.git
if ($LASTEXITCODE -ne 0) { throw 'Clone failed' }
$skillDestination = Join-Path $env:USERPROFILE '.agents\skills\freertos-development-review'
if (Test-Path -LiteralPath $skillDestination) { throw 'Skill already exists; review local changes before updating' }
New-Item -ItemType Directory -Force -Path (Split-Path $skillDestination) | Out-Null
Copy-Item -LiteralPath '.\freertos-development-review\skills\freertos-development-review' -Destination $skillDestination -Recurse
```

Linux / macOS：

```bash
git clone https://github.com/Xylonzzz/freertos-development-review.git && (
  skill_destination="$HOME/.agents/skills/freertos-development-review"
  if [ -e "$skill_destination" ] || [ -L "$skill_destination" ]; then
    echo 'Skill already exists; review local changes before updating' >&2
    exit 1
  fi
  mkdir -p "$HOME/.agents/skills" &&
    cp -R freertos-development-review/skills/freertos-development-review "$skill_destination"
)
```

团队也可以把 `skills/freertos-development-review` 整个目录放到目标工程的 `.agents/skills/` 下并纳入版本管理。最终路径应是 `.agents/skills/freertos-development-review/SKILL.md`，保留相邻的 `references/` 和 `agents/`。

Codex 会检测 Skill 变化；如果未出现，重启 Codex 后再检查。可在支持的界面使用 `$` 选择 Skill，或直接在请求中点名。当前安装与发现方式见 [OpenAI 官方 Skill 文档](https://learn.chatgpt.com/docs/build-skills)。已有版本通过 `~/.codex/skills` 安装的用户，可以继续由对应版本的安装器管理，避免重复安装同名 Skill。

## 使用示例

打开自己的固件工程后，明确希望“开发”“先设计”还是“只读审阅”。下面的功能、周期和容量只是请求示例，不是通用参数。

### 开发功能

```text
$freertos-development-review
为现有 FreeRTOS 工程实现异步参数保存。
StorageTask 已独占 Flash，ControlTask 提交请求不能等待写入完成。
请复用现有驱动和工程结构，补齐队列满、写入失败及完成结果反馈，
并执行本工程可用的构建和相关验证。
```

### 先做设计

```text
$freertos-development-review
先设计采集任务到控制任务的数据交接，不改代码。
控制只需要最新完整测量；需要区分采样失败、保留旧值和数据过期。
请比较消息队列与快照，给出所有权、接口和验证条件。
```

### 只读审阅

```text
$freertos-development-review
只读审阅当前 FreeRTOS 工程，从 CAN 接收中断追踪到参数生效和保存。
重点检查分片完整性、队列满、同步、超时与保存结果。
按影响排序，给出文件/符号证据、建议和最小验证场景。
```

### 修复已知问题

```text
$freertos-development-review
修复采集任务持续超时后仍把旧值发布为有效新数据的问题。
先定位真实调用链，在本次授权范围内实现最小修复，
保留现有消息协议，并用首次采集失败和连续采集失败场景验证。
```

给出工程路径、MCU、内核/API 版本、目标行为及已有构建命令会更有帮助；能够从工程文件核实的信息不必重复粘贴。仅有代码摘录时也可以使用，结论会限于已提供的材料。

## 预期结果与边界

开发结果应说明改了哪些文件、前后行为、设计取舍、集成位置，以及实际执行的检查。审阅发现应包含“证据 → 影响 → 最小建议 → 验证条件”，并保留已有合理设计。

Skill 不提供一份可直接烧录的通用框架，也不保证所有端口、内核版本或产品均已验证。具体 API、优先级、栈单位、超时和恢复动作必须结合目标工程确认。AI 给出的代码与结论仍需要工程验证；构建成功不能代替硬件验收。

只读审阅不会自动转为代码修复。“先设计”保持在设计范围内；普通开发按用户授权和项目规则推进。下载固件、复位、故障注入等硬件操作需要对应的操作授权。

当前已有结构检查、代码摘录审阅试用和异步保存设计试用。这些试用不构成跨平台兼容性测试、自动化评测套件或功能安全认证。

## 仓库结构

```text
freertos-development-review/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
└── skills/
    └── freertos-development-review/
        ├── SKILL.md
        ├── agents/
        │   └── openai.yaml
        └── references/
            ├── development.md
            ├── architecture.md
            ├── reliability.md
            └── verification.md
```

入口 [SKILL.md](skills/freertos-development-review/SKILL.md) 选择工作模式并按需加载资料：

| 文件 | 内容 |
|---|---|
| [development.md](skills/freertos-development-review/references/development.md) | 需求契约、机制选择、模块实现和集成 |
| [architecture.md](skills/freertos-development-review/references/architecture.md) | 任务、数据、资源、同步及调度边界 |
| [reliability.md](skills/freertos-development-review/references/reliability.md) | 数据质量、故障处理、监督和持久化 |
| [verification.md](skills/freertos-development-review/references/verification.md) | 异常场景、证据层级与交付要求 |

## 更新与贡献

在本仓库克隆目录执行 `git pull --ff-only` 获取更新。手动复制安装的副本不会跟随仓库自动更新；请比较差异、保留本地定制后再同步。不要同时保留多个同名安装副本。

欢迎提交有最小复现材料的 Issue 或 PR，尤其是版本差异、遗漏的失败路径，以及会让 Skill 误判的真实案例。提交方式与验证要求见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 来源与许可

最初的工程思路来自一位业界工程师分享的《RTOS_企业级嵌入式框架_FreeRTOS_STM32_Demo.md》。本项目将其中的方法提炼为开发与审阅流程，并结合试用补充同步、返回值、数据质量、故障送达和验证边界。原作者的公开署名和公开链接未提供，因此不作推测；参考资料中的章节号用于记录这份来源。

仓库不分发原教程全文、FreeRTOS 内核或其他第三方实现。原教程和外部资料保留各自权利；本仓库的 MIT 许可不对这些外部材料重新授权。API 细节应查阅 [FreeRTOS 官方文档](https://www.freertos.org/Documentation/00-Overview)，本项目与 FreeRTOS、OpenAI 无官方隶属或背书关系。

本仓库中维护的 Skill 指令和配套文档采用 [MIT License](LICENSE)。
