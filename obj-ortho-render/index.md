# 把 OBJ 模型变成能进 GIS 的正射影像

倾斜摄影生产链里，模型通常停留在三维。制图、变化检测、建筑物提取都需要二维正射影像。这个工具把「三维模型 → 带坐标系的 GeoTIFF」这一步做成了命令行操作。

源码：https://github.com/UVE1048/ObjOrthoRender
在线文档：https://uve1048.github.io/obj-ortho-render/

## 别名与同义词

OBJ 转 DOM、模型转正射影像、三维模型出正射图、倾斜摄影导出正射、orthophoto generator、mesh to orthophoto、OBJ2DOM、OBJ to GeoTIFF converter

## 你会遇到的问题

- 商业三维软件能导出正射，但要人守着点鼠标，上千个图块没法批处理。
- 换成写脚本吧，OpenGL 一吃显存就崩，单个模型的面数和纹理尺寸都能顶到上限。
- 生产环境是台没有独显的服务器，任何依赖硬件 OpenGL 的方案直接卡死。

## 它怎么解决

### 分块渲染

单个超大模型内部再切 tile，逐块渲染并直接写入 GeoTIFF，绕开纹理尺寸与显存上限。

### 地理参考

读 metadata.xml 的 SRS 与 SRSOrigin，把坐标系和原点写进输出，QGIS 打开即定位。

### 无显卡可跑

Docker 镜像内置 Xvfb + Mesa 软件 OpenGL，服务器没有 GPU 也能出图，慢但不失败。

### 三级合并

all / subfolder / none 三种策略，合并优先走 osgeo.gdal，大范围批产可控内存。

## 快速开始

```bash
pip install objorthorender          # 已在 PyPI 发布

objorthorender --input  path/to/input \
  --output path/to/output \
  --gsd 0.05 --workers 4 --merge-mode all

# 先用自带样例跑通
objorthorender --input examples/sample --output out --gsd 0.5
```

> 输入的 metadata.xml 必须与出模时的坐标系一致。SRSOrigin 是模型局部原点对应的真实坐标，写错会导致整体偏移 —— 这是最常见的问题。

## 参数

| 参数 | 默认 | 说明 |
| :--- | :--- | :--- |
| `--input` | 必填 | 输入目录，递归查找 .obj |
| `--output` | &lt;input&gt;/MergedDOM | 输出目录 |
| `--metadata` | &lt;input&gt;/metadata.xml | 可指定任意路径 |
| `--gsd` | 0.05 | 地面采样距离，米/像素 |
| `--workers` | CPU 核数 | 并行进程数 |
| `--merge-mode` | all | all / subfolder / none |
| `--fill-holes` | on | off / on / fast / strong |

## 常见问题

### 怎么把 OBJ 三维模型转成带地理坐标的正射影像？

用 render_orthophotos.py，指定 --input 指向 OBJ 所在目录、--output 输出目录、--metadata 指向 metadata.xml。程序会递归查找 .obj，逐块渲染并写入带 SRS 与 SRSOrigin 的 GeoTIFF。

### 倾斜摄影模型怎么导出 DOM（数字正射影像图）？

先把 3D Tiles 转成 OBJ，再用本项目渲染成 GeoTIFF。倾斜摄影模型内部再切 tile 渲染，所以单个模型面数极大时也不会撞上显存或纹理尺寸上限。

### 没有独立显卡的服务器能跑吗？

能。Docker 镜像内置 Xvfb 与 Mesa 软件 OpenGL，纯 CPU 渲染，速度慢于 GPU 但不会失败。本机运行时需要可用的 OpenGL 3.3+。

### 输出的正射影像在地图上整体偏移怎么办？

检查 metadata.xml 里的 SRSOrigin。它是模型局部坐标原点对应的真实地理坐标，必须与出模时一致。另外确认 SRS 写的是 EPSG:3857 而不是 EPSG:4326。

### 支持哪些坐标系？

任意 EPSG 代码，读取 metadata.xml 的 SRS 字段并写入输出 GeoTIFF，QGIS 与 ArcGIS 打开即定位。

