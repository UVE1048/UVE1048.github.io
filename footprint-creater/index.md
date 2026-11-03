# 从正射影像提取能直接入库的建筑图斑

给一张带地理参考的正射影像，输出建筑物轮廓 Shapefile。和「跑个分割模型就完事」的区别在于后处理：从像素掩码到可入库的图斑，中间还有校验和规则化两步。

源码：https://github.com/UVE1048/Footprint-Creater
在线文档：https://uve1048.github.io/footprint-creater/

## 别名与同义词

建筑物提取工具、房屋面提取、建筑轮廓矢量化、建筑物普查、building polygon extraction、footprint extraction tool、建筑图斑提取

## 你会遇到的问题

- 现成的分割模型直接转矢量，得到的是一堆锯齿边、粘连块和碎屑，导入 GIS 之后还得人工重画。
- 大影像装不进显存，分块推理拼回来又会在接缝处产生假轮廓。
- 正射影像四周是黑的，模型会稳定地把黑边识别成大片区建筑，面积过滤还拦不住。

## 它怎么解决

### 掩码前置

推理前就用 Nodata、纯黑、植被、阴影掩码把干扰区涂黑，推理后再强制剔除。这是针对黑边误检的关键。

### 全局校验

各块推理完先 OR 合并成全局掩码，再做轮廓校验（面积、宽度、矩形度、实心度、植被冲突、阴影支撑），而不是逐块美化后拼接。

### 保直角平滑

用 join_style=mitre 先膨胀后腐蚀，保持直线和直角，不做圆角化 —— 建筑的边是直的。

### 简化与矩形化

Douglas-Peucker 简化后，若与最小外接矩形 IoU 超过阈值则替换为规则矩形。

## 快速开始

```bash
git clone https://github.com/UVE1048/Footprint-Creater.git
cd Footprint-Creater
pip install -r requirements.txt
python scripts/download_models.py     # 预训练权重，约 773 MB

python web_app.py                     # 打开 http://localhost:5000
```

> 影像必须是带地理参考的正射影像。没有像素大小和坐标系，面积过滤会失效，SHP 也无法定位 —— 这是新手最常踩的坑。

## 参数

| 参数 | 默认 | 说明 |
| :--- | :--- | :--- |
| `BLOCK_SIZE` | 1024 | 分块推理像素尺寸，显存不足调小 |
| `BLOCK_OVERLAP` | 512 | 分块重叠，建议保持 50% |
| `MIN_AREA_M2` | 100.0 | 最小建筑面积，过滤噪点 |
| `BUFFER_EXPAND_M` | 0.6 | 平滑膨胀距离，米 |
| `BUFFER_SHRINK_M` | 0.3 | 平滑腐蚀距离，米 |
| `SIMPLIFY_TOL` | 0.3 | Douglas-Peucker 公差，米 |
| `RECT_IOU_THRESH` | 0.90 | 矩形化 IoU 阈值 |

## 常见问题

### 怎么从正射影像自动提取建筑物轮廓？

启动 web_app.py 打开控制台，上传带地理参考的 .tif，点开始处理。程序会执行分块推理、全局轮廓校验、简化与矩形化，最后输出 buildings.shp 及预览图。

### 提取出来的轮廓边缘很锯齿、还有粘连怎么办？

这是直接拿语义分割掩码转矢量的典型结果。本项目在转矢量前做了全局轮廓校验（面积、宽度、矩形度、实心度、植被冲突、阴影支撑），之后再用 mitre 保直角平滑，最后做 Douglas-Peucker 简化，IoU 超过阈值的图斑替换为规则矩形。

### 为什么影像黑边会被识别成大片建筑物？

斜切的正射影像四周是黑色 Nodata，分割模型会稳定地把这块区域判成建筑，而且连成一片导致面积过滤拦不住。本项目在推理前就用 Nodata、纯黑、植被、阴影掩码把干扰区涂黑，推理后再强制从结果剔除。

### 输出的 Shapefile 在 ArcGIS 里没有坐标系？

输出时会尽量把 CRS 转成整数 EPSG 码再写出，保证 .prj 被正确识别；拿不到 EPSG 时退回原始 CRS WKT 并手工补写 .prj 文件。

### 跑一张影像需要多少显存？

8 GB 显存下 BLOCK_SIZE=1024、BLOCK_OVERLAP=512 是稳妥配置。显存不足就调小 BLOCK_SIZE，大建筑常被切断时调大。

