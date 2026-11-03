# 从三维模型到建筑图斑：实景三维数据链路完整流程

倾斜摄影模型如何变成可入库的建筑矢量：正射影像渲染、建筑物轮廓提取、模型训练三个环节的具体命令、关键参数与常见坑。

在线版本：https://uve1048.github.io/guide/

---

UVE · 开源工具

项目
完整流程
GitHub

  
  
    Guide · 实景三维数据链路
    
# 从三维模型到建筑图斑：实景三维数据链路完整流程

    倾斜摄影模型 → 正射影像 → 建筑物轮廓 → 入库。这条路看起来是三个工具串联，
    实际上每一段都有自己的坑，而且大部分坑不会报错，只会在最后的成果里以"位置偏了""边缘不对""图斑没法用"
    的形式暴露出来。下面按生产顺序拆开讲。
    
  

  
## 链路总览

    典型的四段，本文覆盖前三段（第四段是人工质检，无法自动化）：
    
```bash
倾斜摄影模型 / 3D Tiles
        │  ① 换成 OBJ（Cesium ion 导出、Metashape、其他转换工具）
        ▼
   OBJ + metadata.xml
        │  ② ObjOrthoRender：分块渲染 + 写地理参考
        ▼
   带坐标系的 GeoTIFF（DOM）
        │  ③ Footprint-Creater：分割 + 校验 + 规则化
        ▼
   buildings.shp（可入库）
        │  ④ 人工质检：套合、拓扑、属性
```

    如果你的模型已经够准、影像也是自己的传感器，第 ③ 步用现成权重就够了。
    只有当现成模型的精度在你的影像上明显不够时，才需要插入"训练自己的模型"这一段。
  

  
## 环节一：三维模型 → 正射影像

    制图、变化检测、建筑物提取都需要二维影像，但模型是三维的。
    这一步的本质是沿 Z 轴正交投影，同时把模型局部坐标换算回真实地理坐标。
    
```bash
# 先用自带样例确认环境正常（约 60 秒）
python render_orthophotos.py --input examples/sample --output out --gsd 0.5

# 处理自己的模型
python render_orthophotos.py \
  --input  /path/to/obj_tiles \
  --output /path/to/dom \
  --gsd 0.05 \
  --workers 4 \
  --merge-mode all
```

    
### 三个关键点

    
      - GSD 决定后续一切。0.05 m 意味着一个像素代表 5 cm。后面的最小建筑面积、平滑距离、
      简化公差都是以米为单位，进而在像素上体现。GSD 和建筑物提取的参数必须匹配，
      用 0.5 m 的影像配 100 m² 最小面积，会得到一堆碎块。

      - metadata.xml 的 SRSOrigin 是最常见的错误来源。它是模型局部原点对应的真实地理坐标。
      写错不会报错，结果是整幅影像偏移几百米——直到你把结果叠到正射底图上才会发现。出模时记下这个值。

      - 无显卡服务器不是问题。Docker 镜像内置 Xvfb + Mesa 软件 OpenGL，纯 CPU 渲染。
      慢，但批处理场景下稳定比快重要。

    
  

  
## 环节二：正射影像 → 建筑物图斑

    这一步最容易被低估。直接拿语义分割掩码转矢量，得到的是锯齿边、粘连块和碎屑，
    导入 GIS 之后几乎不可用。真正耗时间的是"掩码 → 可入库图斑"之间那段处理。
    
```bash
python web_app.py    # 打开 http://localhost:5000 上传影像

# 或者按下面的参数在界面里配
BLOCK_SIZE       1024     # 分块推理尺寸，按显存调
BLOCK_OVERLAP     512     # 建议保持 50%
MIN_AREA_M2     100.0     # 最小建筑面积
BUFFER_EXPAND_M   0.6     # 平滑膨胀，米
BUFFER_SHRINK_M   0.3     # 平滑腐蚀，通常取膨胀的一半
SIMPLIFY_TOL      0.3     # Douglas-Peucker 公差，米
RECT_IOU_THRESH  0.90     # 与最小外接矩形重合度超过此值则矩形化
```

    
### 三个关键点

    
      - 黑边必须前置处理。斜切的正射影像四周是黑色 Nodata，模型会稳定地把这片判成建筑，
      而且连成一片导致面积过滤拦不住。靠事后过滤是治不好的，只能在推理前把干扰区涂黑。

      - 平滑要用直角连接而不是圆角。建筑的边是直的。用圆弧连接平滑出来的轮廓，
      在 GIS 里做拓扑检查、和地籍数据套合时都对不上。

      - 矩形化阈值别调太低。0.90 意味着只有形状已经很接近矩形才强制替换。
      降到 0.7 会把一些真实的不规则建筑也拍成方块，反而降低可用性。

    
  

  
## 可选环节：现成模型不够准就自己训

    什么情况下该训：高密度城区建筑粘连、屋顶材质差异大导致漏检、
    阴影方向不同导致边界系统性偏移。判断标准很简单——看错误是否有规律。
    随机误差靠加数据，系统性偏差只能靠换模型。
    
```bash
cp prepare/DATA.example.TXT prepare/DATA.TXT   # 填入影像与矢量路径
python prepare/prepare_dataset_multiprocess.py  # 滑窗切片
python train.py                                 # HRNet-W48 + FCNHead
python predict.py --input mosaic.tif \
  --checkpoint work_dirs/hrnet_w48_footprint/iter_40000.pth
```

    标签质量决定上限。矢量与影像套合不准时，模型学到的是系统性边界偏移，
    加再多数据也救不回来。训之前务必抽几处叠加核对。
  

  
## 环节三：结果能不能入库，怎么判断

    交付前至少查这四件事，都能在 QGIS 里几分钟完成：
    
      检查项怎么查不合格的表现
      
        | 坐标系 | 图层属性看 CRS | 显示未知或与底图错位 |

        | 套合精度 | 叠到原影像，抽 10 处看边 | 整体平移或旋转 → 回查 GSD 与 SRSOrigin |

        | 几何有效性 | 拓扑检查器跑自相交 | 有自相交面 → 简化公差可能过大 |

        | 面积分布 | 统计字段直方图 | 出现大量 < 20 m² 图斑 → 最小面积阈值偏低 |

      
    
  

  
## 常见问题速查

    
      正射影像整体偏移了几百米？
        九成是 metadata.xml 的 SRSOrigin 与出模时不一致。其次是 SRS 写成了 EPSG:4326 而实际是 3857。
      输出一片全黑？
        OpenGL 没起来。Docker 下确认 DISPLAY=:99 与 Xvfb；本机跑先执行 glxinfo | grep OpenGL 看有没有 3.3+。
      提取结果边缘锯齿严重？
        说明跳过了规则化，或者 SIMPLIFY_TOL 设得过小。默认 0.3 m 是配合 0.05–0.1 m GSD 的经验值。
      大范围生产时合并阶段内存爆掉？
        改用 --merge-mode subfolder，或用 --subdir 按区块分批处理。all 模式会把全部影像并成一张巨型 TIFF。
      能直接处理 3D Tiles 吗？
        不能，链路第一段需要先把 3D Tiles 转成 OBJ（Cesium ion 导出或第三方转换工具）。本工具从 OBJ 开始。
    
  

  
## 相关项目

    
      ObjOrthoRender
        环节一把 OBJ 模型渲染成带地理参考的 GeoTIFF。
      Footprint-Creater
        环节二从正射影像提取建筑物轮廓，输出 Shapefile。
      HRNet-Footprint-Trainer
        可选环节在自己的影像上训练分割模型。
