+++
title = '复现foundation-pose'
date = 2025-03-30T16:00:00+08:00
lastmod = 2025-03-30T16:00:00+08:00
draft = false
+++

## 用自己的数据集复现foundation-pose

今天用自己的数据集复现了foundation-pose 6d姿态估计算法，记录下复现方法。复现一个开源项目，首先是要研究项目的input文件夹，在本项目中就是demo_data/bottle文件，可以看到该文件下有四个子文件夹，分别是：depth、rgb、masks、mesh，以及一个相机内参文件cam_K.txt。

### 制作depth和rgb文件夹，同步获取cam_K.txt
将下面脚本放在bottle文件夹中，运行即可采集对齐好的rgb和depth图，脚本会同步获取相机内参文件保存在cam_K.txt中。
```
import pyrealsense2 as rs  
import os  
import cv2  
import numpy as np  
  
base_dir = os.getcwd()  
rgb_dir = os.path.join(base_dir, 'rgb')  
depth_dir = os.path.join(base_dir, 'depth')  
  
os.makedirs(rgb_dir, exist_ok=True)  
os.makedirs(depth_dir, exist_ok=True)  
  
pipeline = rs.pipeline()  
config = rs.config()  
config.enable_stream(rs.stream.color, 848, 480, rs.format.bgr8, 30)  
config.enable_stream(rs.stream.depth, 848, 480, rs.format.z16, 30)  
  
profile = pipeline.start(config)  
align = rs.align(rs.stream.color)  
  
# 保存 FoundationPose 需要的 cam_K.txtintr = profile.get_stream(rs.stream.color).as_video_stream_profile().get_intrinsics()  
cam_k_path = base_dir + "/cam_K.txt"  
with open(cam_k_path, "w") as f:  
    f.write(f"{intr.fx:.18e} 0.000000000000000000e+00 {intr.ppx:.18e}\n")  
    f.write(f"0.000000000000000000e+00 {intr.fy:.18e} {intr.ppy:.18e}\n")  
    f.write("0.000000000000000000e+00 0.000000000000000000e+00 1.000000000000000000e+00\n")  
  
# 可选：保存深度尺度  
depth_scale = profile.get_device().first_depth_sensor().get_depth_scale()  
with open("../bottle_435_work/depth_scale.txt", "w") as f:  
    f.write(f"{depth_scale}\n")  
  
idx = 0  
  
try:  
    while True:  
        frames = pipeline.wait_for_frames()  
        frames = align.process(frames)  
  
        color_frame = frames.get_color_frame()  
        depth_frame = frames.get_depth_frame()  
        if not color_frame or not depth_frame:  
            continue  
  
        color_bgr = np.asanyarray(color_frame.get_data())  
        depth = np.asanyarray(depth_frame.get_data())   # uint16, aligned depth  
  
        # 预览  
        depth_vis = cv2.applyColorMap(cv2.convertScaleAbs(depth, alpha=0.03), cv2.COLORMAP_JET)  
        cv2.imshow("color | aligned depth", np.hstack((color_bgr, depth_vis)))  
  
        # 保存：rgb 与 depth 名字严格对应  
        cv2.imwrite(f"rgb/{idx:06d}.png", color_bgr)  
        cv2.imwrite(f"depth/{idx:06d}.png", depth)  
  
        idx += 1  
  
        if cv2.waitKey(1) & 0xFF == ord("q"):  
            break  
  
finally:  
    pipeline.stop()  
    cv2.destroyAllWindows()
```

### 制作masks文件夹
masks下面存放的是视频流的第一帧的mask，我的做法是将采集到的第一个图片放入labelme中，然后用ai画出mask再手动精修，最后导出为json文件后运行下面脚本（脚本放在bottle文件夹中）将其转化为同名的mask图。
```
import json  
import cv2  
import numpy as np  
from pathlib import Path  
  
base_dir = Path(os.getcwd()) / 'masks'  
json_path = list(base_dir.glob("*.json"))[0]  
json_name = json_path.stem  # 不带后缀的文件名  
  
out_path = base_dir / f"{json_name}.png"   # 输出完整文件路径  
  
with open(json_path, "r", encoding="utf-8") as f:  
    data = json.load(f)  
  
h, w = data["imageHeight"], data["imageWidth"]  
mask = np.zeros((h, w), dtype=np.uint8)  
  
for shape in data["shapes"]:  
    pts = np.array(shape["points"], dtype=np.int32)  
    cv2.fillPoly(mask, [pts], 255)  
  
cv2.imwrite(str(out_path), mask)  
print("saved:", out_path)
```
### 制作mesh文件夹
mesh文件是复现foundation-pose时最复杂的一个文件，该文件夹下主要包括三个文件，分别是：
- bottle.obj： 待识别物体的3D模型，我是用iphone-ARCode软件采集的，还可以用他腾讯混元3D、soilderworks建模等方式获取。
- bottle.mtl： obj和texture文件的衔接文件，制定哪个texture附着在obj上。
- textured.png：纹理文件，可通过blender软件获取，方法如下：
	- 导入ARCode软件生成的usdz文件
	- 打开UV编辑，选中以_tex0.png结尾的文件
	- 点击图像  -> 另存为

## 运行
将做好的几个文件夹放入demo_data下，修改run_demo.py文件，然后执行`python run_demo.py`即可运行