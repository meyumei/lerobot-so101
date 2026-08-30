# ACT 训练

## 1. 创建训练环境

```bash
conda create -n lerobot-so101-train python=3.12 pip -y
conda activate lerobot-so101-train
```

检查：

```bash
python --version
which python
```

---

## 2. 安装 PyTorch

```bash
pip install   torch==2.10.0   torchvision==0.25.0   --index-url https://download.pytorch.org/whl/cu128
```

检查：

```bash
python - <<'PY'
import torch
import torchvision

print("PyTorch:", torch.__version__)
print("torchvision:", torchvision.__version__)
print("CUDA runtime:", torch.version.cuda)
print("CUDA available:", torch.cuda.is_available())

if torch.cuda.is_available():
    print("GPU:", torch.cuda.get_device_name(0))
    print("VRAM GB:", torch.cuda.get_device_properties(0).total_memory / 1024**3)
PY
```

---

## 3. 安装 FFmpeg

```bash
conda install -y ffmpeg=7.1.1 -c conda-forge
ffmpeg -version | head -1
```

---

## 4. 克隆并固定 LeRobot 源码

```bash
cd ~
git clone https://github.com/huggingface/lerobot.git lerobot-main
cd ~/lerobot-main
git checkout 4aaff99be4a1d81568c08c8f0296b41b40c99ec4
git rev-parse HEAD
```

安装：

```bash
pip install -e ".[training]"
```

固定 TorchCodec：

```bash
pip install --force-reinstall torchcodec==0.10.0
```

---

## 5. 当前数据规模

```text
Episodes = 30
Frames   = 12660
Batch    = 8
```

近似：

```text
12660 / 8 ≈ 1583 steps / epoch
10000 / 1583 ≈ 6.32 epochs
```

---

## 6. 设置数据路径

### 方法 A：当前实际训练使用的本地高速盘

```bash
cd ~/lerobot-so101
export DATASET_ROOT="$HOME/lerobot-so101/datasets/so101-right-pick-blue-box-v1-70ep"
```

### 方法 B：Featurize 数据集功能

只有在数据集成功添加到实例并实际存在时才使用：

```bash
export DATASET_ROOT="/home/featurize/data/so101-right-pick-blue-box-v1-70ep"
```

二选一，不要同时混用。

---

## 7. 100-step Smoke Test

```bash
CUDA_VISIBLE_DEVICES=0 lerobot-train   --dataset.repo_id=meyumei/so101-right-pick-blue-box-v1-70ep   --dataset.root="$DATASET_ROOT"   --policy.type=act   --policy.device=cuda   --policy.push_to_hub=false   --output_dir=outputs/train/act_pick_blue_box_smoke   --job_name=act_pick_blue_box_smoke   --batch_size=8   --num_workers=4   --steps=100   --save_freq=100   --log_freq=10   --env_eval_freq=0   --wandb.enable=false
```

Smoke test 用来验证：

```text
Dataset
→ DataLoader
→ 两路相机
→ ACT forward
→ loss
→ backward
→ optimizer
→ checkpoint
```

---

## 8. 正式 10k 训练

```bash
CUDA_VISIBLE_DEVICES=0 lerobot-train   --dataset.repo_id=meyumei/so101-right-pick-blue-box-v1-70ep   --dataset.root="$DATASET_ROOT"   --policy.type=act   --policy.device=cuda   --policy.push_to_hub=false   --output_dir=outputs/train/act_pick_blue_box_v1   --job_name=act_pick_blue_box_v1   --batch_size=8   --num_workers=4   --steps=10000   --save_freq=2000   --log_freq=100   --env_eval_freq=0   --wandb.enable=false
```

保存：

```text
002000
004000
006000
008000
010000
```

---

## 9. GPU 监控

另开终端：

```bash
watch -n 1 nvidia-smi
```

---

## 10. 当前第一版训练结果摘要

- 100-step smoke test：成功
- 10000-step 正式训练：成功
- 总训练时间：约 13 分 15 秒
- 后期 loss 进入约 0.19～0.21 附近的小幅波动
- 不能仅凭训练 loss 判断过拟合
- 后续应真机比较 6k / 8k / 10k checkpoint
