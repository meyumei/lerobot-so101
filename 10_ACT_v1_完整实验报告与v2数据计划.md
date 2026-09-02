# SO-101 ACT v1 完整实验报告与 ACT v2 数据计划

> 日期：2026-08-30  
> 项目：SO-101 右臂双视角蓝色物块抓取  
> 阶段：ACT v1 训练与真机验证完成，进入 ACT v2 数据迭代

## 1. 阶段摘要

本阶段已经完整打通：

```text
SO-101 遥操作
→ 双相机示范采集
→ LeRobotDataset 检查
→ 云端 ACT 训练
→ checkpoint 导出
→ Windows 本地模型加载
→ CUDA 真机推理
→ 多次真实场景评估
```

当前最重要的结论：

```text
部署成功 ≠ 抓取成功
```

ACT v1 已经学会根据视觉输入向蓝色物块靠近，但没有稳定学会最后的精细夹爪对准与闭合。008000 在采集时多个典型位置 4 次重点测试中，4/4 会靠近，0/4 完整抓取；010000 仍表现为“会靠近但不夹取”。

因此项目下一阶段不再把主要精力放在 rollout 命令或继续增加训练 step，而是更新示范数据。

## 2. 硬件与环境

### Windows 部署端

```text
Windows 11
GPU: NVIDIA GeForce RTX 4060 Laptop GPU
Python: 3.12.14
LeRobot: 0.6.2
Commit: 4aaff99be4a1d81568c08c8f0296b41b40c99ec4
PyTorch: 2.10.0+cu126
TorchVision: 0.25.0+cu126
TorchCodec: 0.10.0
PyAV: 15.1.0
FFmpeg: 7.1.1
CUDA available: True
```

### 云端训练端

```text
GPU: RTX 4090
Python: 3.12.14
LeRobot: 0.6.2
Commit: 4aaff99be4a1d81568c08c8f0296b41b40c99ec4
PyTorch: 2.10.0+cu128
TorchVision: 0.25.0+cu128
CUDA runtime: 12.8
TorchCodec: 0.10.0
PyAV: 15.1.0
FFmpeg: 7.1.1
```

本地 cu126 与云端 cu128 不完全一致，但实际模型加载和真机推理均成功，说明当前差异不是 ACT v1 抓取失败的主要原因。

## 3. 机器人与相机配置

2026-08-30 部署会话确认：

```text
right follower
  port = COM3
  id   = so101_right_follower

right leader
  port = COM7
  id   = so101_right_leader
```

相机：

```text
overhead     = OpenCV Camera 1
right_wrist  = OpenCV Camera 2
640 x 480
30 FPS
```

> COM 和相机 index 可能因 USB 重新枚举变化；固定 ID 和 camera key 不变。

## 4. ACT v1 数据

实际训练数据：

```text
Episodes = 30
Frames   = 12,660
FPS      = 30
Tasks    = 1
```

输入：

```text
observation.state
observation.images.overhead
observation.images.right_wrist
```

输出：

```text
action
```

State / Action：

```text
6D / 6D
```

任务文本：

```text
Pick up the blue box.
```

注意：数据集 repo_id 名称中包含 `70ep`，但 ACT v1 实际训练数据是 30 episodes。

## 5. ACT v1 训练

正式训练：

```text
Policy            ACT
Batch size        8
Steps             10,000
Approx steps/ep   1,583
Approx epochs     6.32
chunk_size        100
n_action_steps    100
Training time     ~13m15s
```

checkpoints：

```text
002000
004000
006000
008000
010000
```

后期 loss：

```text
约 0.19~0.23 小幅波动
```

训练 loss 本身没有暴露真机精细抓取失败，因此后续必须坚持真机评估而不是只看 loss。

## 6. 模型部署验证

008000 已在 Windows 本地成功：

```text
ACTPolicy.from_pretrained(...)
→ MODEL LOAD OK
→ CUDA
→ 两路相机连接
→ SO-101 Follower 连接
→ sync inference
→ action 下发
→ 真机运动
```

标准 25 秒命令：

```cmd
lerobot-rollout --strategy.type=base --policy.path="F:\260826_so101\models\act_pick_blue_box_export\008000" --robot.type=so101_follower --robot.port=COM3 --robot.id=so101_right_follower --robot.cameras="{overhead: {type: opencv, index_or_path: 1, width: 640, height: 480, fps: 30}, right_wrist: {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30}}" --task="Pick up the blue box." --fps=30 --duration=25 --device=cuda --return_to_initial_position=True
```

## 7. 真机评估结果

### 7.1 008000

部署 smoke：

```text
5 秒：真机正常运动，链路打通
```

15 秒探索：

```text
向蓝色物块靠近
但未成功抓取
```

25 秒系统评估：

```text
采集时多个典型物块位置
共 4 次重点测试
```

结果：

| 项目 | 结果 |
|---|---:|
| 向物块靠近 | 4/4 |
| 夹爪稳定位于物块两侧 | 0/4 |
| 完整抓取 | 0/4 |

典型行为：

```text
向物块靠近
→ 到物块附近
→ 末端在附近调整/徘徊
→ gripper 没有形成正确夹持几何关系
→ 抓取失败
```

### 7.2 010000

默认 `n_action_steps=100`：

```text
仍然会向物块靠近
但没有完成正确夹取
```

结论：训练继续到 10k 没有解决当前核心瓶颈。

### 7.3 `n_action_steps=25`

对 008000 临时覆盖：

```text
--policy.n_action_steps=25
```

行为比默认 100 更差：

```text
没有形成原先稳定的靠近轨迹
```

同时完整模型 forward 更频繁，日志持续出现约 12~14 Hz 慢循环 warning。

因此 ACT v1 后续部署恢复：

```text
n_action_steps=100
```

## 8. 运行性能观察

008000 默认 `n_action_steps=100` 的多次 25 秒实验，整体 effective cadence 约 28 Hz，虽然日志周期性输出低 Hz warning，但整段运行并没有持续降到 4~10 Hz。

典型完整运行：

```text
effective cadence ≈ 28.3~28.6 Hz
```

因此当前抓取失败不应优先归因于 RTX 4060 算力不足。

## 9. Camera 2 偶发超时

一次 25 秒实验在约 3.8 秒中断：

```text
OpenCVCamera(2) latest frame is too old: 523.8 ms
max allowed: 500 ms
```

该错误属于腕部相机帧停止更新，不属于模型推理失败。

之后连续多次 25 秒实验完整结束，因此当前作为独立的 USB/相机稳定性问题继续监控。

## 10. 为什么当前判断为数据瓶颈

当前观察具有高度一致性：

```text
008000：会靠近，但不会正确夹取
010000：会靠近，但不会正确夹取
n_action_steps=25：没有改善，反而更差
```

这说明：

1. 模型已经从数据中学到物块与大范围动作方向之间的关系。
2. 部署环境和动作链路没有阻止模型产生有效 reaching。
3. 失败集中在最后的精细抓取阶段。
4. 单纯从 8k 继续训练到 10k 没有解决问题。

因此当前阶段把主要瓶颈定义为：

```text
ACT v1 demonstration 数量和精细抓取一致性不足
```

这里不只指“30 太少”，还包括：

- 最后 5~10 cm 是否足够慢、足够清楚；
- wrist_roll 是否一致；
- gripper 是否真正从物块两侧进入；
- 闭合时机是否一致；
- 同一初始条件是否存在多种彼此冲突的抓取轨迹；
- 失败示范是否被混入正式数据。

## 11. ACT v2 数据设计原则

### 11.1 保持不变

```text
robot.id = so101_right_follower
teleop.id = so101_right_leader
camera keys = overhead, right_wrist
resolution = 640 x 480
fps = 30
state = 6D
action = 6D
task = Pick up the blue box.
```

不要在第二版同时更换相机布局、机械臂 calibration、任务对象和动作策略，否则难以判断改进来自哪里。

### 11.2 强化最后 5~10 cm

每条成功示范尽量形成一致结构：

```text
远距离 approach
→ 接近物块后减速
→ 调整 wrist / gripper 方向
→ gripper 中心对准物块中心
→ 两指进入物块两侧
→ 闭合
→ 保持
→ 抬升
```

### 11.3 一致性优先于盲目增加数量

避免：

```text
同一位置
有时从左边夹
有时从右边夹
有时 wrist_roll 完全相反
有时早闭合
有时晚闭合
```

应尽量让相似状态对应相似动作。

### 11.4 失败 episode 重录

以下 episode 不进入正式训练集：

- 没抓到；
- 抓偏但强行继续；
- 夹爪碰撞物块或桌面；
- 摄像头掉帧/停帧；
- 严重遮挡；
- 操作者中途犹豫并产生大量无意义来回动作。

### 11.5 位置覆盖

可以继续覆盖多个物块位置，但建议分层：

```text
中心位置：高比例、重复稳定成功
轻微偏移：逐步扩展
边缘位置：确认基础策略稳定后再增加
```

这样先让模型学稳抓取，再扩大泛化范围。

## 12. ACT v2 推荐工作流

```text
制定统一抓取动作规范
→ 录 5~10 条 pilot
→ 逐条检查最后抓取阶段
→ 调整相机/操作方式
→ 正式采集新批次
→ Dataset Load Test
→ 抽查视频与动作
→ 云端 smoke train
→ 正式训练
→ 导出多 checkpoint
→ 固定位置真机评估
→ 再评估位置泛化
```

建议优先使用新的数据集版本名，例如：

```text
so101-right-pick-blue-box-v2
```

不要覆盖 ACT v1，以便后续做严格对比。

## 13. 下一版评估指标

ACT v2 不只记录最终成功/失败，建议拆成：

| 指标 | 含义 |
|---|---|
| Reaching rate | 是否进入目标附近 |
| Alignment rate | 夹爪是否正确位于物块两侧 |
| Grasp rate | 是否成功闭合夹住 |
| Lift rate | 是否成功抬起 |
| Full success rate | 完整任务成功 |
| Camera fault rate | 相机异常次数 |

这样能判断改进具体发生在哪一阶段。

## 14. 当前阶段最终结论

ACT v1 不是“完全没学会”，而是：

```text
粗粒度 reaching 已经学到
精细 grasp alignment 没有学稳
```

本阶段已经完成最重要的工程验证：

```text
数据 → 训练 → checkpoint → 本地加载 → 真机动作
```

下一阶段不需要重新解决部署链路，而应该把注意力集中在示范数据质量上。

项目正式进入：

```text
ACT v2 数据采集与训练阶段
```
