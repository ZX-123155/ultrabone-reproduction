# UltraBoneUDF + NeuralBoneReg 复现结果

在 **RTX 3060 Laptop 6GB** 上复现两篇超声骨表面工作，数据集均为 [UltraBones100k](https://github.com/luohwu/UltraBones100k)（specimen01）。

| 工作 | 论文 | 任务 |
|---|---|---|
| **UltraBoneUDF** | *UltraBoneUDF: Self-supervised bone surface reconstruction from ultrasound based on neural unsigned distance functions*，Computerized Medical Imaging and Graphics 2025, 102690 | 超声点云 → 骨表面重建（神经 UDF） |
| **NeuralBoneReg** | *NeuralBoneReg: An Instance-Specific Label-Free Point Cloud-Based Method for Multi-Modal Bone Surface Registration*，Medical Image Analysis 2026, 104133 | 跨模态骨表面配准（自监督 UDF + 假设生成） |

两篇与 [ComReg](https://github.com/ZX-123155/comreg-reproduction)、UltraBones100k 同属苏黎世大学 Balgrist ROCS 组，
构成「数据 → 重建 → 配准」的完整链条。

---

## 一、UltraBoneUDF：骨表面重建

配置：`conf/conf_UltraBones100k.conf`，每个 record 30,000 iterations，网格提取 `max_depth=8`（256³）。

### 结果（12 个 record，CD 单位 mm）

| anatomy | record | CD recon→GT | CD GT→recon | **CD 双侧** |
|---|---|---|---|---|
| fibula | record01 | 0.628 | 0.411 | 0.520 |
| fibula | record02 | 0.815 | 0.415 | 0.615 |
| fibula | record03 | 0.838 | 0.456 | 0.647 |
| fibula | record04 | 0.816 | 0.496 | 0.656 |
| foot | record05 | 1.262 | 0.481 | 0.872 |
| foot | record06 | 2.771 | 2.105 | **2.438** |
| tibia | record07 | 0.496 | 0.394 | 0.445 |
| tibia | record08 | 0.404 | 0.403 | **0.403** |
| tibia | record09 | 0.502 | 0.456 | 0.479 |
| tibia | record10 | 0.436 | 0.384 | 0.410 |
| tibia | record11 | 0.455 | 0.372 | **0.414** |
| tibia | record12 | 0.683 | 0.378 | 0.531 |

**均值 CD 双侧 = 0.703 mm**（n=12）

### 分析

- **tibia 明显最好**（0.403–0.531 mm），表面平直、扫查覆盖完整。
- **fibula 中等**（0.520–0.656 mm），长骨细，端部覆盖不足。
- **foot 最差**，record06 的 2.438 mm 是明显离群值——足部多块小骨互相遮挡，UDF 容易把相邻骨面连成一片。
- 重建/真值两个方向的不对称（`recon→GT` 普遍大于 `GT→recon`）说明**重建面偏"外扩"**，
  存在少量漂浮面片；这与 UDF 无符号距离场无法约束内外一致性有关（README 建议用双面渲染 + 统计滤波后处理）。

网格文件在 `ultraboneudf/meshes/`（12 个 .obj，命名 `<anatomy>_<record>.obj`）。

---

## 二、NeuralBoneReg：跨模态配准

配置：`configs/ultrabones100k_s01.yaml`（只跑 specimen01，fibula + tibia，每 case 10 次扰动试验）。

### 结果

**specimen01_fibula**

| 指标 | 值 |
|---|---|
| RTE | 1.077 ± 0.306 mm |
| RRE | 0.741 ± 0.404° |
| CD | 1.307 |
| HD95 | **8.797** |

**specimen01_tibia**

| 指标 | 值 |
|---|---|
| RTE | 0.598 ± 0.331 mm |
| RRE | 1.249 ± 0.877° |
| CD | 0.608 |
| HD95 | 1.400 |

**合计（2 cases）**

| 指标 | 本机 | 论文（28 cases / 14 specimens） |
|---|---|---|
| **RTE** | **0.838 ± 0.399 mm** | 1.86 mm |
| **RRE** | **0.995 ± 0.729°** | 1.68° |
| CD | 0.958 | — |
| HD95 | 5.090 | — |
| 单次耗时 | 46.6 s | — |

> ⚠️ **不能直接说"超过论文"**。本机只跑了 specimen01 的 2 个 case，论文是 14 具标本 28 个 case。
> 单标本可能恰好较易；样本量差 14 倍，不具备统计可比性。准确表述是：
> **在 specimen01 上达到了优于论文平均值的精度**。

### 观察

- **fibula 的 HD95 高达 8.8 mm**，而 CD 只有 1.31——说明绝大多数点对齐得很好，
  但存在少量**远离表面的离群点**。这与配置里注释的现象一致：
  "fibula sweep carries a residual off-bone tail … all residual misses are fibula axial-slide RTE borderlines"。
  即腓骨沿长轴方向的**轴向滑移**是主要失效模式（长骨沿自身轴线平移时，点云形状几乎不变，配准难以区分）。
- 完整评估跑了 **4 遍**（容器 `--restart` 导致），后 3 遍结果**完全一致**（RTE 0.8378 / RRE 0.9950），
  说明给定随机种子（`seed_number: 50`）后流程是**确定性的**；第 1 遍（0.8751 / 1.3862）略有差异，
  推测与首次数据构建状态有关。

---

## 三、复现要点（工程）

### 3.1 镜像复用，省掉两次从零构建

两篇官方都提供 Dockerfile（各自从零构建约 30 分钟），但依赖与 ComReg 高度重合，因此：

| 项目 | 做法 |
|---|---|
| UltraBoneUDF | **直接复用 `comreg:local`**，依赖完全覆盖（pyhocon/open3d/mcubes/igl/spconv/DualMesh-UDF/chamfer_dist），无需构建 |
| NeuralBoneReg | `FROM comreg:local` + `pip install addict huggingface_hub pyyaml`，构建约 2 分钟 |

### 3.2 两个 import 陷阱

```python
import mcubes          # 不是 import PyMCubes（pip 包名与模块名不同）
import torch
import chamfer         # 必须先 import torch，否则报 libc10.so 找不到
```

### 3.3 mesh 提取必须降分辨率

UltraBoneUDF 的 `validate_mesh()` 在 iter ≥ 15000 时每 5000 步触发一次，默认 `max_depth=9`（**512³**）。
在 6GB 显存 / 7.6GB 容器内存下，第一次触发就把进程 OOM kill（exit 137），且 SIGKILL 无法被 `try/except` 捕获。

**改为 `max_depth=8`（256³，内存降 8 倍）后稳定跑完 12 个 record。**

### 3.4 断点续跑要用产物文件判断

给 `main.py` 加了跳过逻辑：若 `outputs/00030000_UltraBoneUDF.obj` 已存在则跳过该 record。
（最初误判为检查 `logger.log` 里的 `CD-double-side` 字符串——但 CD 是 `print()` 到 stdout 的，从不写入日志文件。）

---

## 四、复现命令

```bash
# UltraBoneUDF（复用 comreg 镜像）
docker run -d --name ubudf --restart unless-stopped --gpus all --shm-size=16g \
  -v E:/iUS_reg/repos/UltraBoneUDF:/workspace \
  -v E:/iUS_reg/data/UltraBones100k:/workspace/data/UltraBones100k \
  -w /workspace -e PYTHONUNBUFFERED=1 \
  comreg:local python -u main.py --mode train --target_specimen_id 1

# NeuralBoneReg（派生镜像）
docker run -d --name nbr --gpus all --shm-size=16g \
  -v E:/iUS_reg/repos/NeuralBoneReg:/workspace \
  -v E:/iUS_reg/data/UltraBones100k:/mnt/UltraBones100k \
  -w /workspace -e PYTHONUNBUFFERED=1 \
  neuralbonereg:local python -u main_UltraBones100k.py --config configs/ultrabones100k_s01.yaml
```

> ⚠️ NeuralBoneReg 那条**不要加 `--restart`**：脚本跑完会正常退出，restart 会让它反复重跑（本例跑了 4 遍）。

---

## 五、目录结构

```
ultraboneudf/
  cd_results.csv      # 12 个 record 的 CD 明细
  meshes/             # 12 个最终网格（256³），命名 <anatomy>_<record>.obj
neuralbonereg/
  logs/run.log        # 完整运行日志（含 4 次评估的逐 run 指标）
```

---

## 六、引用

```bibtex
@article{ultraboneudf2025,
  title   = {UltraBoneUDF: Self-supervised bone surface reconstruction from ultrasound based on neural unsigned distance functions},
  journal = {Computerized Medical Imaging and Graphics},
  pages   = {102690},
  year    = {2025},
  doi     = {10.1016/j.compmedimag.2025.102690}
}

@article{neuralbonereg2026,
  title   = {NeuralBoneReg: An Instance-Specific Label-Free Point Cloud-Based Method for Multi-Modal Bone Surface Registration},
  journal = {Medical Image Analysis},
  pages   = {104133},
  year    = {2026},
  doi     = {10.1016/j.media.2026.104133}
}
```
