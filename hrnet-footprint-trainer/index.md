# 用你自己的影像训一个建筑分割模型

公开数据集上训好的模型，换到你的城市就未必准。这个仓库把「大影像加矢量标签 → 切片数据集 → 训练 → 大图推理」整条链路补齐了。

源码：https://github.com/UVE1048/HRNet-Footprint-Trainer
在线文档：https://uve1048.github.io/hrnet-footprint-trainer/

## 别名与同义词

建筑物分割训练、HRNet 训练脚本、遥感分割训练模板、建筑物提取模型训练、building segmentation training、HRNet segmentation pipeline

## 你会遇到的问题

- 公开数据集训练的模型，到高密度城区就粘连：楼挨得太近，几栋连成一片。
- 阴影方向不一样，学到的边界会系统性偏移；屋顶材质差异大，彩钢瓦直接漏检。
- 想自己训，却发现切片、标签栅格化、大图推理拼接每一段都要自己写。

## 它怎么解决

### 滑窗切片

以 1024 为窗口、512 为步长对大影像重叠裁切，50% 重叠消除边缘效应。增量式：已存在的切片自动跳过，补数据不用重来。

### 前景占比过滤

建筑像素占比低于 1% 或高于 98% 的切片丢弃。全空和全满样本没信息量，丢掉显著降低训练时间。

### NoData 忽略

把纯黑纯白区域在标签里置为 ignore_index，模型不会因为「把黑边预测成背景」被惩罚。这是全套流程里最见效的改动。

### 重叠滑窗推理

重叠窗口预测后拼接，既绕开显存限制，也用重叠区加权消除拼缝断层。

## 快速开始

```bash
git clone https://github.com/UVE1048/HRNet-Footprint-Trainer.git
cd HRNet-Footprint-Trainer
conda create -n footprint python=3.10 -y && conda activate footprint
conda install -c conda-forge geopandas rasterio -y
pip install -r requirements.txt

cp prepare/DATA.example.TXT prepare/DATA.TXT   # 填入影像与矢量路径
python prepare/prepare_dataset_multiprocess.py
python train.py
```

> 标签质量决定上限。矢量与影像套合不准时，模型学到的是系统性边界偏移，加数据也没用。训之前先抽几处叠加检查。

## 参数

| 参数 | 默认 | 说明 |
| :--- | :--- | :--- |
| `CROP_SIZE` | 1024 | 切片边长，像素 |
| `STRIDE` | 512 | 滑窗步长，即 50% 重叠 |
| `TRAIN_RATIO` | 0.8 | 训练集比例 |
| `MIN_FG_RATIO` | 0.01 | 低于此建筑占比的切片丢弃 |
| `MAX_FG_RATIO` | 0.98 | 高于此建筑占比的切片丢弃 |
| `--crop-size` | 1024 | 推理滑窗边长，与训练一致 |

## 常见问题

### 怎么用自己的数据训练一个建筑物提取模型？

准备正射影像 .tif 与建筑物轮廓 .shp（坐标系必须一致），填进 prepare/DATA.TXT，运行 prepare_dataset_multiprocess.py 生成切片数据集，再运行 train.py 训练，最后用 predict.py 对大图做滑窗推理。

### 影像太大装不进显存怎么办？

训练阶段用 1024 滑窗、512 步长切片；推理阶段用重叠滑窗预测再拼接。两处都靠重叠消除边缘效应与拼缝断层。

### 正射影像的黑边会干扰训练吗？

默认已处理。MaskNoData 变换会把纯黑 (0,0,0) 与纯白 (255,255,255) 区域在标签里置为 ignore_index，算 loss 时完全跳过，模型不会因为「把黑边预测成背景」被惩罚。

### 训练需要多少标注数据？

这项任务的边界精度高度依赖标注质量而非数量。标签与影像套合不准时，模型学到的是系统性边界偏移，加数据也无法纠正。训之前务必抽几处叠加检查。

### mmcv 装不上 / 编译失败怎么办？

换 pip install mmcv-lite>=2.1.0。train.py 里已有一行版本号 hack 绕过 MMSegmentation 的强校验，predict.py 里内置 mmcv.ops 的 polyfill，不需要 CUDA 算子也能推理。

