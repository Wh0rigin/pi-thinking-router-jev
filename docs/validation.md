# 验证结果与复现

本文档记录 `pi-thinking-router-jev` 的完整验证结果。测试设计、mock Jev 协议和各层测试覆盖见[测试设计与运行手册](../test/README.md)。

## 验证分层

| 层级 | 内容 | 依赖 |
|---|---|---|
| 1 | 单元测试：档位、错误分类、状态和策略钳制 | 无，完全离线 |
| 2 | 集成测试：完整决策回路与 mock Jev | 无，完全离线 |
| 3 | live 测试：真实 Jev 协议与判别力 | Jev endpoint 和凭据 |
| 4 | E2E / A·B：真实 pi 会话与 3-seed 消耗对比 | pi、模型和 Jev 凭据 |

## 1. 单元与集成测试

运行：

```bash
npm test
```

离线测试覆盖：

- 档位解析与 pi 档位折叠；非法 Jev 输出不会传入 `setThinkingLevel`；
- 推理型失败与环境型失败分类，包含裸数字 `500` 不得误判为 HTTP 错误等对抗样本；
- 任务类型、工具调用、失败、测试运行和变更文件统计；
- 单步调整、升级/降级上限、任务基线、稳定窗口、pin、冷却和新鲜失败旁路；
- 配置校验、Jev HTTP 失败、畸形回答、未配置、手动接管和关闭开关；
- JSONL 决策日志的字段、顺序、计数和 token 分列。

关键回归场景：

```text
fresh failure bypasses the cooldown left by the task-start call
downgrade right after an escalation is pinned
environment-only failures never escalate nor call jev
jev HTTP failure -> falls back to local rules, keeps routing alive
malformed jev answer -> fallback, invalid level never applied
hung jev endpoint -> timeout fires, falls back to rules, loop completes
```

集成测试用真实 `node:http` mock 服务器按调用顺序返回正常回答、HTTP 5xx 或畸形响应，验证完整曲线：

```text
task-start(medium)
  → failure(high)
  → clean turn
  → stable downgrade(medium)
  → failure(high; xhigh suggestion is clamped)
  → cap reached
  → manual override
```

![测试结果](../assets/test-results.png)

## 2. 脚本化全回路

mock Jev 的脚本化场景验证档位曲线：

```text
low → medium → high → medium → high → high
```

最后一次保持 `high` 是升级上限生效的结果。

![mock loop](../assets/mock-loop.png)

## 3. 真实 Jev 连通性与判别力

```bash
JEV_LIVE=1 JEV_ENDPOINT=... JEV_MODEL=jev-latest npm run test:live
JEV_ENDPOINT=... JEV_MODEL=jev-latest npm run ping
```

真实 `jev-1.13.0` 调用记录显示：

- 琐碎重命名和文档修改主要选择 `low`；
- 普通调试任务主要选择 `medium`；
- 并发 NPE 排查选择 `high`；
- 全新缓存架构设计可选择 `xhigh`；
- 测试失败不一定升级：如果问题明显是一行修复，Jev 可能选择保持 `medium`。

同一状态快照的 Jev 结果可能不同，因此本地钳制层负责保证不越级、不频繁抖动。

![jev distributions](../assets/jev-distributions.png)

![jev latency](../assets/jev-latency.png)

## 4. 真实 pi 端到端

运行：

```bash
pi -e ./index.ts --no-session -p "<task>"
```

已验证三类任务：

| 任务 | 路由结果 | 结果 |
|---|---|---|
| 琐碎变量重命名 | `high → low` | 1 次工具调用完成 |
| 先跑测试再修复 | `high → medium` | 修复成功 |
| 强制先跑失败测试 | 失败触发后咨询 Jev，保持 `medium` | 测试恢复，最终修复 |

![E2E routing](../assets/e2e-routing.png)

E2E 曾发现一个真实问题：短任务中的首个推理失败会落在 task-start 调用留下的 15 秒冷却窗口内。现在新鲜失败会绕过该窗口，回归用例已加入集成测试。

## 5. 3-seed A/B 实验

运行：

```bash
npm run benchmark
```

实验比较固定 `thinking=max` 与 thinking-router，两个分支各运行 3 次；每次使用全新临时目录，A/B 交替执行以降低时间漂移影响。

当前记录的结果如下（`mean ± stdev，n=3`）：

| 指标 | max | thinking-router | 变化 |
|---|---:|---:|---:|
| wall 时间 | 23.2 ± 5.2 s | 22.7 ± 1.0 s | −2%，基本持平 |
| glm input tokens（不含缓存） | 10590 ± 7398 | 9368 ± 5633 | −11.5%，方向性 |
| glm output tokens | 253 ± 46 | 248 ± 43 | −2% |
| 任务成功 | 3/3 | 3/3 | 持平 |

Jev 侧额外消耗约 `2.27k tokens/任务`（约 2130 input + 138 output，4 次决策）。Jev 与主模型价格不同，必须分开统计，不能直接用 Jev token 抵扣主模型 token。

![3-seed A/B](../assets/ab-3seed.png)

结论：本次小样本实验中成功率没有下降，wall 时间基本持平，主模型 input token 有下降趋势但方差较大；`n=3` 只能作为方向性信号，不能视为稳定基准。

原始数据位置：`tmp/benchmark-results.json`。图表重新生成：

```bash
npm run charts
```

## 6. 限制与实验口径

- LLM API 没有真正的 seed；“3-seed”指 3 次独立重复。
- 当前实验使用的 `glm-5.3-flash` 只有 `max → max` 的有效思考参数映射，其余档位不发思考参数；因此对比本质是“每轮 max”与“多数轮不带思考参数 + Jev 开销”。
- pi 不直接暴露相关文件，`relevant_files` 由 read/grep 与 edit/write 路径去重近似。
- `task_type` 是本地关键词启发式分类，最终档位仍由 Jev 和本地策略层共同决定。
- `thinking_level_select` 是通知型事件，无法阻止用户手动改档。
- print 模式看不到 TUI 状态栏徽标，应以决策日志确认路由结果。

实验环境：pi 0.86.1、glm-5.3-flash、jev-1.13.0、Node 24、Windows。
