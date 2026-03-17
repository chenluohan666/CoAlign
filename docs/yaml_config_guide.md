# CoAlign YAML配置文件详解

本文档详细说明了项目中所有YAML配置文件的作用、用途和使用场景。

---

## 目录

- [配置文件目录结构](#配置文件目录结构)
- [按数据集分类](#按数据集分类)
  - [OPV2V数据集](#opv2v数据集)
  - [DAIR-V2X数据集](#dairv2x数据集)
  - [V2X-Sim 2.0数据集](#v2xsim-20数据集)
- [按方法分类](#按方法分类)
- [配置文件详解](#配置文件详解)
  - [单智能体检测方法](#1-单智能体检测方法)
  - [早期融合方法](#2-早期融合方法)
  - [中间融合方法](#3-中间融合方法)
  - [晚期融合方法](#4-晚期融合方法)
  - [CoAlign专用配置](#5-coalign专用配置)
  - [相机感知方法](#6-相机感知方法)
  - [可视化配置](#7-可视化配置)
- [配置文件核心参数说明](#配置文件核心参数说明)

---

## 配置文件目录结构

```
opencood/hypes_yaml/
├── opv2v/                          # OPV2V数据集配置
│   ├── lidar_only_with_noise/      # LiDAR方法（含噪声）
│   │   ├── coalign/                # CoAlign相关配置
│   │   └── *.yaml                  # 其他方法配置
│   ├── camera_no_noise/            # 相机方法配置
│   └── visualization_opv2v.yaml    # 可视化配置
├── dairv2x/                        # DAIR-V2X数据集配置
│   ├── lidar_only_with_noise/
│   └── visualization_dair.yaml
└── v2xsim/                         # V2X-Sim 2.0数据集配置
    ├── lidar_only_with_noise/
    └── visualization.yaml
```

---

## 按数据集分类

### OPV2V数据集
- **路径**: `opencood/hypes_yaml/opv2v/`
- **特点**: 仿真数据集，左手法则坐标系，检测范围 [-140.8m, 140.8m] × [-40m, 40m]
- **适用场景**: 车对车(V2V)协作感知研究

### DAIR-V2X数据集
- **路径**: `opencood/hypes_yaml/dairv2x/`
- **特点**: 真实世界数据集，右手法则坐标系，检测范围 [-100m, 100m] × [-40m, 40m]
- **适用场景**: 车路协同(V2I)协作感知研究

### V2X-Sim 2.0数据集
- **路径**: `opencood/hypes_yaml/v2xsim/`
- **特点**: 仿真数据集，右手法则坐标系，检测范围 [-32m, 32m] × [-32m, 32m]
- **适用场景**: V2X协作感知研究

---

## 按方法分类

| 融合类型 | 方法名 | 配置文件 |
|---------|--------|----------|
| 单智能体 | PointPillar Single | `pointpillar_single.yaml` |
| 单智能体 | SECOND Single | `SECOND.yaml` |
| 早期融合 | Cooper (Early Fusion) | `pointpillar_early.yaml`, `SECOND_early.yaml` |
| 中间融合 | F-Cooper | `pointpillar_fcooper.yaml` |
| 中间融合 | Attentive Fusion | `pointpillar_selfatt.yaml` |
| 中间融合 | V2VNet | `pointpillar_v2vnet.yaml` |
| 中间融合 | V2X-ViT | `pointpillar_v2xvit.yaml` |
| 中间融合 | MASH | `pointpillar_mash.yaml` |
| 中间融合 | DiscoNet | `pointpillar_disconet.yaml` |
| 中间融合 | V2VNet (Robust) | `pointpillar_v2vnet_robust.yaml` |
| 中间融合 | FPV-RCNN | `fpvrcnn.yaml` |
| 中间融合 | FVoxel-RCNN | `fvoxelrcnn.yaml` |
| 中间融合 | **CoAlign** | `coalign/pointpillar_coalign.yaml` |

---

## 配置文件详解

### 1. 单智能体检测方法

#### `pointpillar_single.yaml`

**用途**: 单智能体PointPillar基线模型

**目的**: 
- 作为协作感知的性能下界参考
- 验证协作带来的性能提升

**特点**:
- 仅使用自车传感器数据
- 采用Late Fusion数据加载方式（`fusion.core_method: 'late'`）
- `only_vis_ego: true` 仅可视化自车

**训练命令**:
```bash
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/pointpillar_single.yaml
```

**推理命令**:
```bash
python opencood/tools/inference.py --model_dir <model_path> --fusion_method no
```

---

#### `SECOND.yaml`

**用途**: 单智能体SECOND检测器

**目的**:
- 提供基于稀疏卷积的单智能体检测基线
- 体素尺寸更小 (0.1m vs 0.4m)，检测更精细

**特点**:
- 使用3D稀疏卷积 (spconv)
- 体素尺寸: [0.1, 0.1, 0.1]
- 检测范围较小: [-72m, 72m] × [-48m, 48m]

**模型架构**:
```yaml
model:
  core_method: second_ssfa
  args:
    mean_vfe:        # 均值体素特征编码
    spconv:          # 稀疏卷积backbone
    map2bev:         # 转换到鸟瞰图
    ssfa:            # 空间语义特征聚合
    head:            # 检测头
```

---

### 2. 早期融合方法

#### `pointpillar_early.yaml`

**用途**: 早期融合（Early Fusion）基线方法，又称Cooper

**目的**:
- 将所有智能体的原始点云合并后统一处理
- 作为协作感知的性能上界（理想情况）
- 可作为DiscoNet的教师模型

**特点**:
- `fusion.core_method: 'early'`
- `fusion.args.proj_first: true` - 先变换点云坐标再合并
- 最大CAV数量: 5

**数据流**:
```
各CAV点云 → 坐标变换到自车坐标系 → 合并 → 统一编码 → 检测
```

**训练命令**:
```bash
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/pointpillar_early.yaml
```

**注意事项**:
- 通信带宽需求大（传输原始点云）
- 对位姿误差敏感

---

#### `SECOND_early.yaml`

**用途**: 基于SECOND的早期融合方法

**目的**: 与PointPillar早期融合对比，验证不同backbone对融合效果的影响

**特点**: 
- 使用SECOND作为backbone
- 体素尺寸更小，适合精细检测

---

### 3. 中间融合方法

#### `pointpillar_fcooper.yaml`

**用途**: F-Cooper方法实现

**论文**: [F-Cooper](https://arxiv.org/abs/1909.06459) (SEC 2019)

**目的**: 
- 使用最大值池化融合多智能体特征
- 简单有效的中间融合基线

**特点**:
- `fusion_method: max` - 最大值融合
- 不添加噪声训练（可用于对比）
- 训练周期较短 (20 epochs)

**融合策略**:
```yaml
model:
  args:
    fusion_method: max  # 逐元素取最大值
```

**训练命令**:
```bash
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/pointpillar_fcooper.yaml
```

---

#### `pointpillar_selfatt.yaml`

**用途**: Attentive Fusion方法实现

**论文**: [Where2comm](https://arxiv.org/abs/2209.12836) / Attentive Fusion

**目的**:
- 使用自注意力机制融合多智能体特征
- 学习智能体间的重要性权重

**特点**:
- 多尺度特征融合
- 注意力权重学习

**模型架构**:
```yaml
model:
  core_method: point_pillar_intermediate
  args:
    base_bev_backbone:
      layer_nums: [3, 5, 8]
      num_filters: [64, 128, 256]
      upsample_strides: [1, 2, 4]  # 多尺度上采样
```

---

#### `pointpillar_selfatt_singlescale.yaml`

**用途**: 单尺度Attentive Fusion

**目的**: 与多尺度版本对比，验证多尺度融合的有效性

**特点**: 
- 仅使用单一尺度特征
- 计算量较小

---

#### `pointpillar_v2vnet.yaml`

**用途**: V2VNet方法实现

**论文**: [V2VNet](https://arxiv.org/abs/2008.07519) (ECCV 2020)

**目的**:
- 使用图神经网络进行智能体间特征传播
- ConvGRU进行多轮消息传递

**特点**:
- `fusion_method: v2vnet`
- 2轮迭代传播 (`num_iteration: 2`)
- 使用ConvGRU进行时序建模

**模型配置**:
```yaml
model:
  args:
    fusion_method: v2vnet
    v2vnet:
      num_iteration: 2          # 消息传递轮数
      in_channels: 256
      gru_flag: true            # 使用GRU
      agg_operator: "max"       # 聚合方式
      conv_gru:
        H: 50
        W: 176
        num_layers: 1
        kernel_size: [[3,3]]
```

**训练命令**:
```bash
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/pointpillar_v2vnet.yaml
```

---

#### `pointpillar_v2vnet_robust.yaml`

**用途**: V2VNet (Robust) 方法 - 抗位姿误差版本

**论文**: [V2VNet Robust](https://arxiv.org/abs/2011.05289) (CoRL 2020)

**目的**: 
- 带位姿回归的抗噪声方法
- 学习预测位姿误差并校正

**特点**:
- 需要三阶段训练 (`stage: 0/1/2`)
- 包含位姿回归模块
- 需要GT位姿监督

**三阶段训练配置**:
```yaml
# Stage 0: 基础训练
stage: &stage 0
train_params:
  epoches: 15
optimizer:
  lr: 0.001
lr_scheduler:
  step_size: [10]

# Stage 1: 位姿回归训练  
stage: 1
epoches: 20
lr: 0.002
step_size: [10, 20]

# Stage 2: 端到端微调
stage: 2
epoches: 30
lr: 0.0002
step_size: [10, 20]
```

**模型架构**:
```yaml
model:
  core_method: point_pillar_v2vnet_robust
  args:
    v2vfusion:        # V2V特征融合
    robust:           # 抗噪声模块
      feature_dim: 256
      hidden_dim: 256
      downsample_rate: 4
    stage: *stage     # 当前训练阶段
```

**训练流程**:
```bash
# Stage 0: 从V2VNet预训练开始
python opencood/tools/train.py --hypes_yaml ... --model_dir <v2vnet_pretrain>

# Stage 1 & 2: 需要修改yaml中的stage参数后继续训练
```

---

#### `pointpillar_v2xvit.yaml`

**用途**: V2X-ViT方法实现

**论文**: [V2X-ViT](https://arxiv.org/abs/2203.02144) (ECCV 2022)

**目的**:
- 使用Vision Transformer进行多智能体特征融合
- 多尺度窗口注意力机制
- 对位姿误差有一定鲁棒性

**特点**:
- Agent-wise注意力（智能体间）
- Spatial-wise注意力（空间窗口）
- 3层编码器，每层多尺度窗口

**模型配置**:
```yaml
model:
  args:
    fusion_method: v2xvit
    v2xvit:
      transformer:
        encoder:
          num_blocks: 1          # 每层block数
          depth: 3               # 编码器层数
          use_roi_mask: true     # 使用ROI mask
          cav_att_config:        # Agent-wise注意力
            dim: 256
            heads: 8
            dim_head: 32
            dropout: 0.3
          pwindow_att_config:    # 多尺度窗口注意力
            heads: [16, 8, 4]
            window_size: [4, 8, 16]
```

**训练命令**:
```bash
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/pointpillar_v2xvit.yaml
```

---

#### `pointpillar_mash.yaml`

**用途**: MASH方法实现

**论文**: [MASH](https://arxiv.org/abs/2107.00771) (IROS 2021)

**目的**:
- 学习像素级对应关系
- 避免使用噪声位姿

**特点**:
- 需要batch_size=1训练
- 构建相似性体积找对应关系
- 训练周期较长 (50 epochs)

**模型架构**:
```yaml
model:
  core_method: point_pillar_mash
  args:
    mash:
      feature_dim: 256
      query_dim: 32    # Query向量维度
      key_dim: 32      # Key向量维度
      H: 50
      W: 176
      downsample_rate: 4
```

**损失函数**:
```yaml
loss:
  core_method: point_pillar_mash_loss
  args:
    cls_weight: 1.0
    grid_weight: 1.0  # 对应关系损失
    reg: 2.0
```

---

#### `pointpillar_disconet.yaml`

**用途**: DiscoNet方法实现

**论文**: [DiscoNet](https://arxiv.org/abs/2111.00643) (NeurIPS 2021)

**目的**:
- 使用知识蒸馏训练轻量级协作模型
- 从Early Fusion教师模型学习

**特点**:
- 需要先训练Early Fusion教师模型
- 使用`train_w_kd.py`训练
- 知识蒸馏损失权重较大

**配置关键项**:
```yaml
kd_flag:
  teacher_model: point_pillar_disconet_teacher
  teacher_model_config: *model_args
  teacher_path: "opencood/logs/opv2v_point_pillar_lidar_early_xxx/net_epoch_bestval_at29.pth"

loss:
  args:
    kd:
      weight: 10000  # 蒸馏损失权重
```

**训练流程**:
```bash
# Step 1: 训练教师模型
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/pointpillar_early.yaml

# Step 2: 修改pointpillar_disconet.yaml中的teacher_path

# Step 3: 使用KD训练
python opencood/tools/train_w_kd.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/pointpillar_disconet.yaml
```

---

#### `fpvrcnn.yaml`

**用途**: FPV-RCNN方法实现

**论文**: [FPV-RCNN](https://arxiv.org/abs/2109.11615) (RAL 2022)

**目的**:
- 两阶段检测器
- 使用点特征细化检测结果

**特点**:
- 两阶段训练 (`activate_stage2: False/True`)
- 使用VSA (Voxel Set Abstraction) 模块
- ROI Head进行框细化

**模型架构**:
```yaml
model:
  core_method: fpvrcnn
  args:
    activate_stage2: False  # Stage1关闭Stage2
    mean_vfe:              # 体素特征编码
    spconv:                # 稀疏卷积
    map2bev:               # 转换到BEV
    ssfa:                  # 空间语义特征聚合
    head:                  # Stage1检测头
    vsa:                   # 体素集抽象
      num_keypoints: 4096
      features_source: ['bev', 'x_conv1', 'x_conv2', 'x_conv3', 'x_conv4', 'raw_points']
    roi_head:              # Stage2 ROI头
      grid_size: 6
```

**两阶段训练**:
```bash
# Stage 1: 仅训练第一阶段
# 设置 activate_stage2: False
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/fpvrcnn.yaml

# Stage 2: 启用第二阶段
# 设置 activate_stage2: True, epoches改为40
python opencood/tools/train.py --hypes_yaml ... --model_dir <stage1_checkpoint>
```

---

#### `fvoxelrcnn.yaml`

**用途**: FVoxel-RCNN方法实现

**目的**: 
- 类似FPV-RCNN的两阶段检测器
- Stage2需要batch_size=1

**特点**: 与FPV-RCNN类似，但Stage2配置略有不同

---

### 4. 晚期融合方法

晚期融合配置文件主要通过修改`fusion.core_method: 'late'`实现，如`pointpillar_single.yaml`。

**特点**:
- 各智能体独立检测
- 在输出层面融合（NMS）
- 对位姿误差最不敏感

---

### 5. CoAlign专用配置

#### `coalign/pointpillar_coalign.yaml`

**用途**: CoAlign主配置文件 - 完整方法

**论文**: [CoAlign](https://arxiv.org/abs/2211.07214) (ICRA 2023)

**目的**:
- 实现抗位姿误差的协作感知
- 使用Agent-Object位姿图优化校正位姿
- 多尺度特征融合

**特点**:
- 混合融合策略（中间+晚期）
- 位姿图优化模块
- 不需要GT位姿监督

**关键配置**:
```yaml
# 噪声设置
noise_setting:
  add_noise: True
  args:
    pos_std: 0.2   # 位置噪声标准差(米)
    rot_std: 0.2   # 旋转噪声标准差(度)

# Box Alignment配置
box_align:
  train_result: "opencood/logs/coalign_precalc/opv2v/train/stage1_boxes.json"
  val_result: "opencood/logs/coalign_precalc/opv2v/val/stage1_boxes.json"
  test_result: "opencood/logs/coalign_precalc/opv2v/test/stage1_boxes.json"
  args:
    use_uncertainty: true       # 使用不确定性权重
    landmark_SE2: true          # SE(2)位姿图
    adaptive_landmark: false
    normalize_uncertainty: false
    abandon_hard_cases: true
    drop_hard_boxes: true

# 多尺度融合模型
model:
  core_method: point_pillar_baseline_multiscale
  args:
    fusion_method: att
    att:
      feat_dim: [64, 128, 256]  # 多尺度特征维度
```

**训练流程**:
```bash
# 需要先准备好预计算的stage-1 boxes
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/coalign/pointpillar_coalign.yaml
```

---

#### `coalign/pointpillar_coalign_woba.yaml`

**用途**: CoAlign无Box Alignment版本 (woba = without box alignment)

**目的**:
- 消融实验：验证Box Alignment模块的贡献
- 仅使用多尺度特征融合

**特点**:
- 注释掉了`box_align`配置
- 其他与完整CoAlign相同

**配置差异**:
```yaml
# box_align被注释
# box_align:
#   train_result: "..."
#   val_result: "..."
#   test_result: "..."
```

---

#### `coalign/pointpillar_uncertainty.yaml`

**用途**: 带不确定性估计的PointPillar单智能体检测器

**目的**:
- 为CoAlign预计算Stage-1 Boxes
- 输出检测框及其不确定性

**特点**:
- `core_method: point_pillar_uncertainty`
- 不确定性维度: 3 (x, y, yaw)
- 使用Von-Mise损失处理角度不确定性

**不确定性损失配置**:
```yaml
loss:
  core_method: point_pillar_uncertainty_loss
  args:
    uncertainty:
      weight: 0.25
      angle_weight: 0.5
      dim: 3
      xy_loss_type: l2        # 位置使用L2损失
      angle_loss_type: von-mise  # 角度使用Von-Mise损失
      lambda_V: 0.001
      s0: 1
```

**训练命令**:
```bash
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/coalign/pointpillar_uncertainty.yaml
```

---

#### `coalign/SECOND_uncertainty.yaml`

**用途**: 基于SECOND的不确定性估计模型

**目的**: 
- 使用更强的backbone (SECOND) 预计算Stage-1 Boxes
- 检测精度更高

**特点**:
- 体素尺寸更小 (0.1m)
- 使用稀疏卷积
- 检测范围更大 [-140.8m, 140.8m]

---

#### `coalign/precalc.yaml`

**用途**: 预计算Stage-1 Boxes的配置文件

**目的**:
- 使用已训练的不确定性模型
- 生成train/val/test集的检测框JSON文件

**关键配置**:
```yaml
box_align_pre_calc:
  stage1_model: second_ssfa_uncertainty
  stage1_model_config:     # Stage1模型配置
    lidar_range: ...
    voxel_size: [0.1, 0.1, 0.1]
    uncertainty_dim: 3
  stage1_model_path: "opencood/logs/opv2v_SECOND_uncertainty_xxx/net_epoch_bestval_at31.pth"
  stage1_postprocessor_name: uncertainty_voxel_postprocessor
  output_save_path: "opencood/logs/coalign_precalc/opv2v"
```

**使用命令**:
```bash
python opencood/tools/pose_graph_pre_calc.py --hypes_yaml opencood/hypes_yaml/opv2v/lidar_only_with_noise/coalign/precalc.yaml
```

---

### 6. 相机感知方法

#### `lss_coalign_fusion.yaml`

**用途**: 基于相机的CoAlign方法

**目的**:
- 使用Lift-Splat-Shoot进行相机BEV特征提取
- 应用CoAlign的多尺度融合策略

**特点**:
- `input_source: ['camera']`
- 使用4个环视相机
- EfficientNet作为图像编码器

**关键配置**:
```yaml
fusion:
  args:
    grid_conf:
      xbound: [-48, 48, 0.4]
      ybound: [-48, 48, 0.4]
      ddiscr: [2, 50, 48]  # 深度离散化
    data_aug_conf:
      cams: ['camera0', 'camera1', 'camera2', 'camera3']
      Ncams: 4

model:
  core_method: lift_splat_shoot_intermediate
  args:
    img_features: 128
    camera_encoder: EfficientNet
    fusion_args:
      core_method: att_ms  # 多尺度注意力融合
```

**训练命令**:
```bash
python opencood/tools/train.py --hypes_yaml opencood/hypes_yaml/opv2v/camera_no_noise/lss_coalign_fusion.yaml
```

---

#### `lss_v2xvit.yaml`

**用途**: 基于相机的V2X-ViT方法

**特点**: 将V2X-ViT的Transformer融合策略应用于相机BEV特征

---

#### `lss_v2vnet_fusion.yaml`

**用途**: 基于相机的V2VNet方法

**特点**: 将V2VNet的图神经网络融合策略应用于相机BEV特征

---

#### `lss_selfatt.yaml`

**用途**: 基于相机的自注意力融合方法

---

#### `lss_single_resnet101.yaml` / `lss_single_efficientnet.yaml`

**用途**: 单智能体相机检测基线

**目的**: 对比不同图像backbone (ResNet101 vs EfficientNet) 的性能

---

### 7. 可视化配置

#### `visualization_opv2v.yaml` / `visualization_dair.yaml` / `visualization.yaml`

**用途**: 专用于数据可视化的配置

**目的**:
- 可视化数据集中的点云和标注
- 不用于训练或推理

**特点**:
- 名称为"visualization"
- 不包含训练相关参数
- 通常使用late fusion加载数据

**使用场景**:
```python
# 在可视化脚本中使用
hypes = yaml_utils.load_yaml('opencood/hypes_yaml/opv2v/visualization_opv2v.yaml', opt)
dataset = build_dataset(hypes, visualize=True, train=False)
```

---

## 配置文件核心参数说明

### noise_setting（噪声设置）

```yaml
noise_setting:
  add_noise: True           # 是否添加噪声
  args:
    pos_std: 0.2            # 位置噪声标准差(米)
    rot_std: 0.2            # 旋转噪声标准差(度)
    pos_mean: 0             # 位置噪声均值
    rot_mean: 0             # 旋转噪声均值
```

**说明**:
- 用于模拟定位系统的位姿误差
- 训练时添加噪声可提高模型鲁棒性
- `add_noise: False` 表示不添加噪声（用于无噪声场景对比）

### fusion（融合设置）

```yaml
fusion:
  core_method: 'intermediate'  # 融合类型: early/late/intermediate/intermediate2stage
  dataset: 'opv2v'
  args:
    proj_first: false          # 是否先投影点云(2-round通信)
```

**融合类型说明**:
| core_method | 说明 |
|-------------|------|
| `early` | 早期融合，合并原始点云 |
| `late` | 晚期融合，合并检测结果 |
| `intermediate` | 中间融合，合并特征图 |
| `intermediate2stage` | 两阶段中间融合(FPV-RCNN用) |

### train_params（训练参数）

```yaml
train_params:
  batch_size: 4       # 批大小
  epoches: 30         # 训练轮数
  eval_freq: 2        # 验证频率(epoch)
  save_freq: 3        # 保存频率(epoch)
  max_cav: 5          # 最大CAV数量
```

### preprocess（预处理）

```yaml
preprocess:
  core_method: 'SpVoxelPreprocessor'  # 体素化方法
  args:
    voxel_size: [0.4, 0.4, 4]          # 体素尺寸[x,y,z]
    max_points_per_voxel: 32           # 每体素最大点数
    max_voxel_train: 32000             # 训练时最大体素数
    max_voxel_test: 70000              # 测试时最大体素数
  cav_lidar_range: [-140.8, -40, -3, 140.8, 40, 1]  # 检测范围
```

### model（模型）

```yaml
model:
  core_method: point_pillar_baseline  # 模型核心方法
  args:
    fusion_method: v2vnet             # 融合方法
    # ... 其他模型特定参数
```

### loss（损失函数）

```yaml
loss:
  core_method: point_pillar_loss
  args:
    pos_cls_weight: 2.0               # 正样本分类权重
    cls:                              # 分类损失
      type: 'SigmoidFocalLoss'
      weight: 1.0
    reg:                              # 回归损失
      type: 'WeightedSmoothL1Loss'
      weight: 2.0
    dir:                              # 方向损失
      type: 'WeightedSoftmaxClassificationLoss'
      weight: 0.2
```

### optimizer & lr_scheduler（优化器与学习率调度）

```yaml
optimizer:
  core_method: Adam
  lr: 0.002
  args:
    eps: 1e-10
    weight_decay: 1e-4

lr_scheduler:
  core_method: multistep
  gamma: 0.1
  step_size: [10, 15]  # 学习率衰减的epoch
```

---

## 总结

本文档详细介绍了CoAlign项目中所有YAML配置文件的用途和特点。用户可根据以下指南选择合适的配置：

1. **基线对比**: 使用 `pointpillar_single.yaml` (单智能体) 和 `pointpillar_early.yaml` (早期融合)
2. **SOTA方法对比**: 选择对应的配置文件 (V2VNet, V2X-ViT等)
3. **抗噪声研究**: 使用 `pointpillar_coalign.yaml` 或 `pointpillar_v2vnet_robust.yaml`
4. **相机感知**: 使用 `camera_no_noise/` 目录下的配置
5. **消融实验**: 使用 `coalign/pointpillar_coalign_woba.yaml`

如有问题，请参考[GitHub Issues](https://github.com/yifanlu0227/CoAlign/issues)。
