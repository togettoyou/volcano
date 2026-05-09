# Volcano 学习路线（深度参与版）

> 分支：`learning/roadmap`
> 目标：从零到能在 Volcano 社区提交有质量的 PR、参与设计讨论。
> 使用方式：每完成一项把 `[ ]` 改成 `[x]`，并在末尾追加一句"我学到了什么/卡在哪"。

---

## 阶段 0 · 前置知识（1–2 周）

不补这些会处处卡壳。

- [x] **Kubernetes 调度器基础**
  - [x] kube-scheduler 主循环 `Schedule()` 走一遍
  - [x] 理解 Scheduling Framework 扩展点：PreFilter / Filter / Score / Reserve / Permit / Bind
  - [x] Pod 生命周期、Informer/Lister 机制
  - [x] CRD + Controller 模式（写过一次最好）
- [x] **Go 必备**
  - [x] interface / goroutine / channel / context
  - [x] Go Modules（`go mod tidy / vendor`）
  - [x] 能读懂反射代码（不需会写）
- [x] **批处理调度术语**（每个能用一句话说清）
  - [x] Gang Scheduling
  - [x] DRF（Dominant Resource Fairness）
  - [x] Fair Share / Proportion / Capacity
  - [x] Backfill
  - [x] Preemption vs Reclaim 的区别

---

## 阶段 1 · 跑起来 + 看清边界（第 1 周）

目标：先理解 Volcano 对外提供什么，再看实现。

- [x] 通读 `README.md`
- [x] 本地跑通 Volcano（任选其一）
  - [ ] `hack/local-up-volcano.sh`（Linux/WSL2）
  - [ ] KIND + `kubectl apply -f installer/volcano-development.yaml`
  - [x] 实验环境(controlplane + node01)直接 kubectl apply
- [x] 跑示例
  - [x] `example/` 下最简单的 vcjob (hello-volcano)
  - [ ] `example/integrations/tensorflow` 或 `mpi`
- [x] 列出全部 CRD：`kubectl get crd | grep volcano`
  - [x] 能说清 `Job` / `PodGroup` / `Queue` / `JobFlow` / `JobTemplate` / `HyperNode` 各自作用
- [ ] 用 `vcctl` 提交 / 取消 / 列举 job
- [x] **里程碑**：白板能画出 "用户提交 vcjob → 谁创建 PodGroup → 谁创建 Pod → 谁调度 → 谁绑定" 的完整链路

---

## 阶段 2 · 精读三份核心设计文档（第 2 周）

`docs/design/` 有 70+ 篇，先只读这 3 篇打底。

- [ ] `docs/design/job-api.md` — Volcano Job 生命周期、与原生 Job 的差异
- [ ] `docs/design/execution-flow.md` — 调度器单次 Session 内循环
- [ ] `docs/design/queue/queue.md` — 队列模型、Capacity vs Proportion

学完顺手记一句话总结到本文件末尾。

---

## 阶段 3 · 读控制器（第 3 周）

控制器比调度器简单 10 倍，从这里入门最稳。

调用链按顺序读：

- [ ] `cmd/controller-manager/main.go`
- [ ] `cmd/controller-manager/app/server.go` — 启动入口
- [ ] `pkg/controllers/framework/` — 控制器框架
- [ ] `pkg/controllers/job/`（重点）
  - [ ] `job_controller.go` — 主循环
  - [ ] `job_controller_actions.go` — 创建/同步/删除
  - [ ] `state/` — 状态机
  - [ ] `plugins/` — ssh/svc/env 插件
- [ ] `pkg/controllers/podgroup/` — 给原生 pod 自动建 PodGroup
- [ ] `pkg/controllers/queue/` — 队列状态机
- [ ] **里程碑**：脑内复现 `kubectl create -f vcjob.yaml` 之后的全流程：Job controller 接到事件 → 创建 PodGroup → 按 minAvailable 创建 Pods → 调度器调度 → Pod 状态回写 → Job 状态机推进

---

## 阶段 4 · 读调度器（第 4–6 周，**核心**）

按依赖顺序，**绝对不要直接跳进 plugins**。

### 4.1 调度框架（2–3 天）

- [ ] `pkg/scheduler/scheduler.go` — `runOnce` / Session 入口
- [ ] `pkg/scheduler/framework/session.go` — Session 是核心抽象
- [ ] `pkg/scheduler/framework/job_info.go` — Job/Task/PodGroup 内部模型
- [ ] `pkg/scheduler/framework/plugins.go` — 插件注册与回调点
- [ ] `pkg/scheduler/cache/cache.go` — 调度器自己的缓存（从 Informer 同步）
- [ ] `pkg/scheduler/api/` — 内部 API（JobInfo/TaskInfo/NodeInfo）
- [ ] 弄清 Session 的全部扩展点：
  - [ ] `AddJobOrderFn`
  - [ ] `AddTaskOrderFn`
  - [ ] `AddPredicateFn`
  - [ ] `AddBatchNodeOrderFn`
  - [ ] `AddPreemptableFn`
  - [ ] `AddReclaimableFn`
  - [ ] `AddJobReadyFn` / `AddJobPipelinedFn`

### 4.2 Actions（2 天）

- [ ] `pkg/scheduler/actions/enqueue/` — PodGroup pending → inqueue
- [ ] `pkg/scheduler/actions/allocate/`（先看这个）— 主分配
- [ ] `pkg/scheduler/actions/backfill/` — 给 BestEffort 兜底
- [ ] `pkg/scheduler/actions/preempt/` — 同队列内抢占
- [ ] `pkg/scheduler/actions/reclaim/` — 跨队列回收
- [ ] `pkg/scheduler/actions/shuffle/` — 重平衡

每个 action 文件 200–500 行，把它和 `execution-flow.md` 对上。

### 4.3 Plugins（先吃透 5 个，其余按需）

- [ ] `gang/` — ★必读：Volcano 的灵魂
- [ ] `priority/` — ★必读：最简单的 plugin，看注册套路
- [ ] `drf/` — ★必读：公平共享算法
- [ ] `proportion/` — ★必读：queue 配额
- [ ] `predicates/` — ★必读：复用 k8s scheduler framework
- [ ] `binpack/` 或 `nodeorder/` — 评分类
- [ ] `deviceshare/` — GPU 共享（社区关注度高）
- [ ] `capacity/` — 新一代队列模型（趋势替代 proportion）
- [ ] `network-topology-aware/` — 大模型训练热点
- [ ] `numaaware/` / `task-topology/`

**里程碑**：白板上能画出 `allocate.Execute()` 中，某个 task 怎样依次过 gang → drf → priority → predicates → nodeorder 的回调链。

---

## 阶段 5 · 第一个 PR（第 7 周起）

按难度递增。

- [ ] 在 GitHub 上 watch 仓库，订阅 PR/issue 邮件，看一周真实变更
- [ ] 加入社区
  - [ ] CNCF Slack `#volcano` 频道
  - [ ] 双周社区会议（README 有 Zoom 链接）
  - [ ] 订阅 mailing list
- [ ] 找第一个 issue（按推荐顺序尝试）
  - [ ] 文档 / 注释 / typo（label：`good first issue`、`kind/documentation`）
  - [ ] 单测覆盖率补强：`go test -cover ./pkg/scheduler/plugins/xxx`
  - [ ] 小 bug 修复（label：`help wanted`）
  - [ ] 加一个 metric（`pkg/scheduler/metrics/` + 某个 plugin）
  - [ ] 小 feature（先在 issue 区讨论再动手）
- [ ] 提 PR 前的本地检查
  - [ ] `make verify`
  - [ ] `make unit-test`
  - [ ] `make e2e`（视改动范围）
- [ ] PR 描述里 @ 对应模块的 reviewer（`MAINTAINERS.md` / OWNERS）
- [ ] **里程碑**：合入 1 个 PR

---

## 阶段 6 · 深度参与（数月起）

- [ ] Review 别人的 PR（成为 reviewer 是 maintainer 的前置）
- [ ] 在社区会议提案 / 演示
- [ ] 写一份 design doc 放进 `docs/design/`
- [ ] 认领 SIG（Scheduling / Job-API / Agent-Colocation）
- [ ] 出现在 `MAINTAINERS.md` 或 OWNERS

---

## 立刻可以做的 3 件事

- [ ] 起一套本地集群跑 demo
- [ ] `docs/design/execution-flow.md` + `pkg/scheduler/actions/allocate/allocate.go` 对照读一遍
- [ ] watch 仓库，订阅邮件

---

## 学习日志（追加式）

> 每完成一个阶段，写 1–3 行：今天学到的关键点 / 没看懂的地方。卡住的问题攒着下次问。

- _2026-05-09_ — 阶段 0 完成。K8s 基础（informer / controller / scheduling framework）、Go 概念、批调度术语全部过关；EASY-Backfill 与 Reclaim 的具体场景已补齐。下一步：阶段 1 起本地集群。
- _2026-05-09_ — 阶段 1 主体完成。集群部署 OK；hello-volcano 跑通,亲眼验证 vcjob → PodGroup → Pod 派生关系（owner=vcjob,Pod 通过 annotation 关联 PodGroup）。关键纠偏:vcjob.status 由 Job controller 写而非 scheduler（K8s Single Writer 原则）。理解 PodGroup 抽象的价值:让 scheduler 与具体 Job 类型解耦,生态 (Spark/TF/MPI/Ray) 都通过创建 PodGroup 接入。剩 vcctl 一项后续补。
