# UVE · 三维 GIS 与遥感影像处理开源工具

三维 GIS、摄影测量与遥感影像处理的开源工具：OBJ 模型转正射影像、正射影像建筑物轮廓提取、建筑物分割模型训练、无人机视频融合投射。全部 MIT 开源。

站点：https://uve1048.github.io/
GitHub：https://github.com/UVE1048

## 项目

### ObjOrthoRender — OBJ 三维模型转带地理参考的正射影像

倾斜摄影生产链里，模型通常停留在三维。制图、变化检测、建筑物提取都需要二维正射影像。这个工具把「三维模型 → 带坐标系的 GeoTIFF」这一步做成了命令行操作。

- 源码：https://github.com/UVE1048/ObjOrthoRender
- 文档：https://uve1048.github.io/obj-ortho-render/
- 纯文本：https://uve1048.github.io/obj-ortho-render/index.md
- 别名：OBJ 转 DOM、模型转正射影像、三维模型出正射图、倾斜摄影导出正射、orthophoto generator、mesh to orthophoto、OBJ2DOM、OBJ to GeoTIFF converter

### Footprint-Creater — 正射影像自动提取建筑物轮廓

给一张带地理参考的正射影像，输出建筑物轮廓 Shapefile。和「跑个分割模型就完事」的区别在于后处理：从像素掩码到可入库的图斑，中间还有校验和规则化两步。

- 源码：https://github.com/UVE1048/Footprint-Creater
- 文档：https://uve1048.github.io/footprint-creater/
- 纯文本：https://uve1048.github.io/footprint-creater/index.md
- 别名：建筑物提取工具、房屋面提取、建筑轮廓矢量化、建筑物普查、building polygon extraction、footprint extraction tool、建筑图斑提取

### HRNet-Footprint-Trainer — 在自有影像上训练建筑物分割模型

公开数据集上训好的模型，换到你的城市就未必准。这个仓库把「大影像加矢量标签 → 切片数据集 → 训练 → 大图推理」整条链路补齐了。

- 源码：https://github.com/UVE1048/HRNet-Footprint-Trainer
- 文档：https://uve1048.github.io/hrnet-footprint-trainer/
- 纯文本：https://uve1048.github.io/hrnet-footprint-trainer/index.md
- 别名：建筑物分割训练、HRNet 训练脚本、遥感分割训练模板、建筑物提取模型训练、building segmentation training、HRNet segmentation pipeline

### VideoFusion-Web — 无人机视频融合投射到三维实景

与其在角落放个小窗让操作员心算对应关系，不如把画面按相机位姿投影到模型表面 —— 画面和地物处在同一空间，遮挡、朝向、覆盖范围全部自洽。

- 源码：https://github.com/UVE1048/VideoFusion-Web
- 文档：https://uve1048.github.io/video-fusion-web/
- 纯文本：https://uve1048.github.io/video-fusion-web/index.md
- 别名：视频融合示例、无人机视频投影、视频贴模型、视频投射 SDK 用法、video projection onto 3D model、UAV video fusion
