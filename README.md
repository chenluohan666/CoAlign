# CoAlign (ICRA2023)

Robust Collaborative 3D Object Detection in Presence of Pose Errors 

[Paper](https://arxiv.org/abs/2211.07214) | [Video](https://www.youtube.com/watch?v=zCjpFkeC2rA) | [Readme in Feishu](https://udtkdfu8mk.feishu.cn/docx/LlMpdu3pNoCS94xxhjMcOWIynie) 

![Original1](images/coalign.jpg)

## Update 2024.1.24
**HEAL** is accepted to ICLR 2024. We implement a unified and integrated multi-agent collaborative perception framework for LiDAR-based, camera-based and heterogeneous setting! See [HEAL GitHub](https://github.com/yifanlu0227/HEAL).

## Update 2023.7.11

**Camera-based collaborative perception support!**

We release the multi-agent camera-based detection code, based on [Lift-Splat-Shoot](https://github.com/nv-tlabs/lift-splat-shoot). Support OPV2V, V2XSet and DAIR-V2X-C dataset. 

LiDAR's feature map fusion method can seamlessly adapt to camera BEV feature. Support CoAlign's multiscale fusion, V2XViT, V2VNet, Self-Att, FCooper, DiscoNet(w.o. KD). Please feel free to browse our repo. Example yamls are listed in this folder: `opencood/hypes_yaml/opv2v/camera_no_noise` 

---

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Data Preparation](#data-preparation)
- [Quick Start](#quick-start)
- [Training](#training)
- [Inference & Evaluation](#inference--evaluation)
- [Configuration Files](#configuration-files)
- [Complemented Annotations for DAIR-V2X-C](#complemented-annotations-for-dair-v2x-c)
- [Checkpoints](#checkpoints)
- [Citation](#citation)

---

## Features

### Modality Support
- LiDAR
- Camera

### Dataset Support
- OPV2V
- V2X-Sim 2.0
- DAIR-V2X
- V2XSet

### SOTA Collaborative Perception Methods
| Method | Paper | Config File |
|--------|-------|-------------|
| Attentive Fusion | [ICRA2022](https://arxiv.org/abs/2109.07644) | `pointpillar_selfatt.yaml` |
| Cooper | [ICDCS](https://arxiv.org/abs/1905.05265) | `pointpillar_early.yaml` |
| F-Cooper | [SEC2019](https://arxiv.org/abs/1909.06459) | `pointpillar_fcooper.yaml` |
| V2VNet | [ECCV2022](https://arxiv.org/abs/2008.07519) | `pointpillar_v2vnet.yaml` |
| FPV-RCNN | [RAL2022](https://arxiv.org/pdf/2109.11615.pdf) | `fpvrcnn.yaml` |
| DiscoNet | [NeurIPS2021](https://arxiv.org/abs/2111.00643) | `pointpillar_disconet.yaml` |
| V2X-ViT | [ECCV2022](https://github.com/DerrickXuNu/v2x-vit) | `pointpillar_v2xvit.yaml` |
| MASH | [IROS2021](https://arxiv.org/abs/2107.00771) | `pointpillar_mash.yaml` |
| V2VNet (robust) | [CoRL2020](https://arxiv.org/abs/2011.05289) | `pointpillar_v2vnet_robust.yaml` |
| **CoAlign** | [ICRA2023](https://arxiv.org/abs/2211.07214) | `coalign/pointpillar_coalign.yaml` |

### Other Features
- BEV visualization & 3D visualization
- 1-round/2-round communication support
- Pose error simulation support

---

## Installation

Please visit the feishu docs [CoAlign Installation Guide](https://udtkdfu8mk.feishu.cn/docx/LlMpdu3pNoCS94xxhjMcOWIynie) for details!

Or you can refer to [OpenCOOD data introduction](https://opencood.readthedocs.io/en/latest/md_files/data_intro.html)
and [OpenCOOD installation](https://opencood.readthedocs.io/en/latest/md_files/installation.html) guide to prepare
data and install CoAlign. The installation is totally the same as OpenCOOD, except some dependent packages required by CoAlign.

```bash
# Create conda environment
conda env create -f environment.yml
conda activate coalign

# Install the package
pip install -e .

# Install spconv (for sparse convolution)
pip install spconv-cu117  # or spconv-cu118 based on your CUDA version
```

---

## Data Preparation

Create a `dataset` folder under CoAlign root directory:

```bash
mkdir dataset
```

Put your OPV2V, V2X-Sim, V2XSet, DAIR-V2X data in this folder. The structure should be:

```
CoAlign/dataset
├── my_dair_v2x 
│   ├── v2x_c
│   ├── v2x_i
│   └── v2x_v
├── OPV2V
│   ├── additional
│   ├── test
│   ├── train
│   └── validate
├── V2XSET
│   ├── test
│   ├── train
│   └── validate
├── v2xsim2-complete
│   ├── lidarseg
│   ├── maps
│   ├── sweeps
│   └── v1.0-mini
└── v2xsim2_info
    ├── v2xsim_infos_test.pkl
    ├── v2xsim_infos_train.pkl
    └── v2xsim_infos_val.pkl
```

**Notes:**
1. `*.pkl` files in `v2xsim2_info` can be found in [Google Drive](https://drive.google.com/drive/folders/16_KkyjV9gVFxvj2YDCzQm1s9bVTwI0Fw?usp=sharing)
2. Use our complemented annotation for DAIR-V2X in `my_dair_v2x` (see below)

---

## Quick Start

### Step 1: Download Pre-computed Stage-1 Boxes (for CoAlign)

CoAlign requires pre-computed single-agent detection boxes for pose graph optimization:

```bash
# Download from Google Drive and save to opencood/logs
# https://drive.google.com/drive/folders/1otDzESlepuhRBE4ZgJQfpArnpG1TG8uu
```

### Step 2: Train CoAlign

```bash
# Single GPU training
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/coalign/pointpillar_coalign.yaml

# Multi GPU training (4 GPUs)
CUDA_VISIBLE_DEVICES=0,1,2,3 python -m torch.distributed.launch --nproc_per_node=4 --use_env opencood/tools/train_ddp.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/coalign/pointpillar_coalign.yaml
```

### Step 3: Evaluate

```bash
# Evaluate without additional noise
python opencood/tools/inference.py --model_dir opencood/logs/opv2v_coalign_withnoise_2023_xx_xx --fusion_method intermediate

# Evaluate with various noise levels
python opencood/tools/inference_w_noise.py --model_dir opencood/logs/opv2v_coalign_withnoise_2023_xx_xx --fusion_method intermediate
```

---

## Training

### Basic Training Commands

#### Single GPU Training

```bash
python opencood/tools/train.py --hypes_yaml <CONFIG_FILE> [--model_dir <CHECKPOINT_DIR>]
```

**Arguments:**
- `--hypes_yaml, -y`: Path to the configuration YAML file (required)
- `--model_dir`: Path to continue training from a checkpoint (optional)
- `--fusion_method, -f`: Fusion method for inference (default: `intermediate`)

#### Multi GPU Training (Distributed Data Parallel)

```bash
CUDA_VISIBLE_DEVICES=0,1,2,3 python -m torch.distributed.launch --nproc_per_node=4 --use_env opencood/tools/train_ddp.py --hypes_yaml <CONFIG_FILE> [--model_dir <CHECKPOINT_DIR>] [--half]
```

**Additional Arguments:**
- `--half`: Enable half precision training (FP16)

### Training Examples by Method

#### 1. Single Agent Detection (Baseline)

```bash
# PointPillar single agent
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/pointpillar_single.yaml

# SECOND single agent
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/SECOND.yaml
```

#### 2. CoAlign (Two-Stage Training)

**Stage 1: Generate pre-computed boxes**

First, train the uncertainty estimation model and pre-calculate boxes:

```bash
# Train single-agent detector with uncertainty
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/coalign/pointpillar_uncertainty.yaml

# Pre-calculate stage-1 boxes for train/val/test sets
python opencood/tools/pose_graph_pre_calc.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/coalign/precalc.yaml
```

**Stage 2: Train CoAlign fusion**

```bash
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/coalign/pointpillar_coalign.yaml
```

#### 3. V2VNet (Robust) - Three Stage Training

V2VNet (robust) requires 3-stage training:

```bash
# Stage 0: Train from V2VNet pretrain (set stage: 0, epoches: 15 in yaml)
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/pointpillar_v2vnet_robust.yaml --model_dir <v2vnet_pretrain_path>

# Stage 1: Continue training (set stage: 1, epoches: 20, lr: 0.002)
python opencood/tools/train.py --hypes_yaml ... --model_dir <stage0_checkpoint>

# Stage 2: Final stage (set stage: 2, epoches: 30, lr: 0.0002)
python opencood/tools/train.py --hypes_yaml ... --model_dir <stage1_checkpoint>
```

#### 4. DiscoNet - Knowledge Distillation Training

```bash
# Step 1: Train the teacher model (early fusion)
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/pointpillar_early.yaml

# Step 2: Set teacher path in pointpillar_disconet.yaml

# Step 3: Train with knowledge distillation
python opencood/tools/train_w_kd.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/pointpillar_disconet.yaml
```

#### 5. FPV-RCNN / FVoxel-RCNN

```bash
# Stage 1: Train first stage only (activate_stage2: False)
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/fpvrcnn.yaml

# Stage 2: Train both stages (activate_stage2: True, resume from stage1)
python opencood/tools/train.py --hypes_yaml ... --model_dir <stage1_checkpoint>
```

#### 6. Camera-based Methods

```bash
# CoAlign for camera
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/camera_no_noise/lss_coalign_fusion.yaml

# V2X-ViT for camera
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/camera_no_noise/lss_v2xvit.yaml
```

---

## Inference & Evaluation

### Standard Inference (No Additional Noise)

```bash
python opencood/tools/inference.py --model_dir <MODEL_DIR> --fusion_method <METHOD> [--save_vis_interval N] [--save_npy]
```

**Arguments:**
- `--model_dir`: Path to the model checkpoint directory (required)
- `--fusion_method`: Fusion method, options: `single`, `no`, `no_w_uncertainty`, `early`, `late`, `intermediate` (default: `intermediate`)
- `--save_vis_interval`: Save visualization every N frames (default: 40)
- `--save_npy`: Save prediction and GT results as npy files
- `--no_score`: Do not print prediction scores

### Inference with Noise Simulation

For evaluating robustness under different noise levels:

```bash
python opencood/tools/inference_w_noise.py --model_dir <MODEL_DIR> --fusion_method <METHOD> [--also_laplace]
```

**Additional Arguments:**
- `--also_laplace`: Also evaluate with Laplace noise distribution (useful for DAIR-V2X)

This will evaluate the model under noise levels: `(0, 0), (0.2, 0.2), (0.4, 0.4), (0.6, 0.6)` for both position (meters) and rotation (degrees).

### Example Commands

```bash
# Evaluate CoAlign on OPV2V
python opencood/tools/inference.py --model_dir opencood/logs/opv2v_coalign_withnoise_2023_xx_xx --fusion_method intermediate

# Evaluate with noise variations
python opencood/tools/inference_w_noise.py --model_dir opencood/logs/opv2v_coalign_withnoise_2023_xx_xx --fusion_method intermediate

# Evaluate on DAIR-V2X with Laplace noise
python opencood/tools/inference_w_noise.py --model_dir opencood/logs/dairv2x_coalign_xx --fusion_method intermediate --also_laplace

# Single agent evaluation (no collaboration)
python opencood/tools/inference.py --model_dir <MODEL_DIR> --fusion_method no

# Late fusion evaluation
python opencood/tools/inference.py --model_dir <MODEL_DIR> --fusion_method late
```

---

## Configuration Files

### Directory Structure

```
opencood/hypes_yaml/
├── opv2v/                          # OPV2V dataset configs
│   ├── lidar_only_with_noise/      # LiDAR with pose noise
│   │   ├── coalign/                # CoAlign specific configs
│   │   │   ├── pointpillar_coalign.yaml      # Main CoAlign config
│   │   │   ├── pointpillar_coalign_woba.yaml # CoAlign without box alignment
│   │   │   ├── pointpillar_uncertainty.yaml  # Uncertainty estimation model
│   │   │   ├── precalc.yaml                  # Pre-calculation config
│   │   │   └── SECOND_uncertainty.yaml       # SECOND with uncertainty
│   │   ├── pointpillar_single.yaml           # Single agent baseline
│   │   ├── pointpillar_early.yaml            # Early fusion (Cooper)
│   │   ├── pointpillar_fcooper.yaml          # F-Cooper
│   │   ├── pointpillar_v2vnet.yaml           # V2VNet
│   │   ├── pointpillar_v2vnet_robust.yaml    # V2VNet (robust)
│   │   ├── pointpillar_v2xvit.yaml           # V2X-ViT
│   │   ├── pointpillar_selfatt.yaml          # Attentive Fusion
│   │   ├── pointpillar_mash.yaml             # MASH
│   │   ├── pointpillar_disconet.yaml         # DiscoNet
│   │   ├── fpvrcnn.yaml                      # FPV-RCNN
│   │   ├── fvoxelrcnn.yaml                   # FVoxel-RCNN
│   │   ├── SECOND.yaml                       # SECOND single agent
│   │   └── SECOND_early.yaml                 # SECOND early fusion
│   ├── camera_no_noise/            # Camera configs (no noise)
│   │   ├── lss_coalign_fusion.yaml           # CoAlign for camera
│   │   ├── lss_v2xvit.yaml                   # V2X-ViT for camera
│   │   ├── lss_v2vnet_fusion.yaml            # V2VNet for camera
│   │   ├── lss_selfatt.yaml                  # Self-attention for camera
│   │   └── lss_single_*.yaml                 # Single agent camera
│   └── visualization_opv2v.yaml              # Visualization config
├── dairv2x/                        # DAIR-V2X dataset configs
│   └── lidar_only_with_noise/      # Same structure as OPV2V
├── v2xsim/                         # V2X-Sim 2.0 dataset configs
│   └── lidar_only_with_noise/      # Same structure as OPV2V
└── readme.md
```

### Key Configuration Parameters

#### Data Paths
```yaml
root_dir: "dataset/OPV2V/train"      # Training data path
validate_dir: "dataset/OPV2V/validate" # Validation data path
test_dir: "dataset/OPV2V/test"       # Test data path
```

#### Noise Settings
```yaml
noise_setting:
  add_noise: True                    # Whether to add pose noise
  args:
    pos_std: 0.2                     # Position noise std (meters)
    rot_std: 0.2                     # Rotation noise std (degrees)
    pos_mean: 0                      # Position noise mean
    rot_mean: 0                      # Rotation noise mean
```

#### Training Parameters
```yaml
train_params:
  batch_size: 4                      # Batch size
  epoches: 30                        # Number of epochs
  eval_freq: 2                       # Validation frequency
  save_freq: 3                       # Checkpoint saving frequency
  max_cav: 5                         # Maximum number of CAVs
```

#### Fusion Settings
```yaml
fusion:
  core_method: 'intermediate'        # Fusion type: early, late, intermediate
  dataset: 'opv2v'
  args:
    proj_first: false                # Whether to project point cloud first (2-round comm)
```

#### Box Alignment (CoAlign specific)
```yaml
box_align:
  train_result: "path/to/train_boxes.json"  # Pre-computed stage-1 boxes
  val_result: "path/to/val_boxes.json"
  test_result: "path/to/test_boxes.json"
  args:
    use_uncertainty: true            # Use uncertainty for weighting
    landmark_SE2: true               # Use SE(2) for landmarks
    adaptive_landmark: false
    normalize_uncertainty: false
    abandon_hard_cases: true
    drop_hard_boxes: true
```

#### Model Settings
```yaml
model:
  core_method: point_pillar_baseline_multiscale  # Model architecture
  args:
    voxel_size: [0.4, 0.4, 4]        # Voxel size [x, y, z]
    lidar_range: [-140.8, -40, -3, 140.8, 40, 1]
    fusion_method: att               # Feature fusion method
```

---

## Complemented Annotations for DAIR-V2X-C

Originally DAIR-V2X only annotates 3D boxes within the range of camera's view in vehicle-side. We supplement the missing 3D box annotations to enable 360 degree detection.

Original | Complemented
---|---
![Original1](images/dair-v2x_compare_gif/before1.gif) | ![Complemented1](images/dair-v2x_compare_gif/after1.gif)
![Original2](images/dair-v2x_compare_gif/before2.gif) | ![Complemented2](images/dair-v2x_compare_gif/after2.gif)
![Original3](images/dair-v2x_compare_gif/before3.gif) | ![Complemented3](images/dair-v2x_compare_gif/after3.gif)

**Download:** [Google Drive](https://drive.google.com/file/d/13g3APNeHBVjPcF-nTuUoNOSGyTzdfnUK/view?usp=sharing)

**Website:** [DAIR-V2X-C Complemented](https://siheng-chen.github.io/dataset/dair-v2x-c-complemented/)

---

## Checkpoints

### Single Detection with Uncertainty
Download [coalign_precalc](https://drive.google.com/drive/folders/1otDzESlepuhRBE4ZgJQfpArnpG1TG8uu) and save to `opencood/logs`

### CoAlign Checkpoints
- [CoAlign OPV2V](https://drive.google.com/drive/folders/14VdGUZ26j4NF0UG_XuNsU7E0uGh_DmOU?usp=sharing)
- [CoAlign V2X-Sim 2.0](https://drive.google.com/drive/folders/1ymKGFdto8HECKZFJJbCSkiuWlY6HF7OZ?usp=sharing)
- [CoAlign DAIR-V2X](https://drive.google.com/drive/folders/1zsgEMTGpB_Llz66SqeegXXKdaduIegXE?usp=sharing)

Download and save to `opencood/logs`

---

## Citation

```bibtex
@inproceedings{lu2023robust,
  title={Robust collaborative 3d object detection in presence of pose errors},
  author={Lu, Yifan and Li, Quanhao and Liu, Baoan and Dianati, Mehrdad and Feng, Chen and Chen, Siheng and Wang, Yanfeng},
  booktitle={2023 IEEE International Conference on Robotics and Automation (ICRA)},
  pages={4812--4818},
  year={2023},
  organization={IEEE}
}
```

---

## Acknowledgments

This project is impossible without the code of [OpenCOOD](https://github.com/DerrickXuNu/OpenCOOD), [g2opy](https://github.com/uoip/g2opy) and [d3d](https://github.com/cmpute/d3d)!

Thanks again to [@DerrickXuNu](https://github.com/DerrickXuNu) for the great code framework.

---

## Q&A

1. **Different AP results between arxiv v2 and arxiv v3? and different from OPV2V[ICRA 22']?**

   See [Issue 4](https://github.com/yifanlu0227/CoAlign/issues/4).
   
2. **How to get V2X-Sim-2.0 pickle file?**

   See [Issue 2](https://github.com/yifanlu0227/CoAlign/issues/2).