# yolov5-fastapi-deploy
Record problems encountered during deployment and their solutions

## 所谓API部署

在非生产环境中，无需考虑种种限制，只需在有合适算力的电脑上配置好环境即可本地部署。但在实际工程化的生产环境中，跨平台调用、高并发和系统解耦的需求要求使用API或者其他非本地部署方法。

在模型本身训练好后，将模型文件转为中间表示，再使用推理引擎进行剪枝和量化，即可封装为API进行使用了

## YOLOv5的基本框架

YOLO系列属于目标检测算法，使用标注数据进行有监督学习。

整体架构分为Input(输入处理)、Backbone(主干网络)、Neck(特征融合)、Head(检测头)

### Input

> 在输入端的处理主要包括Mosaic数据增强、自适应锚框计算、自适应图片缩放、随机仿射变换、HSV 颜色空间增强、图像模糊、MixUp和随机水平翻转

YOLOv5和YOLOv4同样在Input部分使用了Mosaic数据增强，其目的是提高小目标的AP(平均精度)、丰富数据集以增强鲁棒性、提高GPU效率等。具体而言，Mosaic数据增强将四张图片随机缩放、裁剪、排布并最终拼接成一张图片作为输入

自适应锚框计算指的是使用K-means聚类算法对标注中所有锚框的尺寸基于IoU(交并比)分为K=9个不同尺寸的锚框簇，并按检测目标的大小分配

自适应图片缩放减少了图像经过统一缩放之后形成的黑边。先将长宽等比例缩小，再在短边按算法补充黑边

### Backbone

> 由Focus结构、CBL模块、CSP模块和SPPF模块构成

Focus结构相较于传统卷积下采样，可以减少计算量并保留更多原图信息;其将输入隔行隔列拆分为四个子图(切片)，再将四个子图在通道维度拼接成一个图(拼接)，最后再进行卷积操作

CBL模块是指Conv+BN+SiLU，是YOLO系列的一个基础模块

CSP模块可以减少计算量和参数量，现更新为C3模块，
CSP1_X模块将输入按照通道维度均分为两路：一路经过CBL和多个残差，再经过一个卷积，再和只经过一个卷积的另一路拼接后通过一个卷积
CSP2_X模块与CSP1_X类似，只是将多个残差换成多个CBL模块

SPPF模块是多尺度池化层，将输入连续多次池化，并将每次池化的结果合并输出;相较于多个池化层串联，SPPF可以有更小的计算量

### Neck

> Neck层没有引入更多的新模块，其主要目的是融合Backbone输出的不同层级的特征图，整合各层的信息并输出多尺度的融合特征图

FPN层自顶向下传递强语义特征

然后添加两个PAN结构构成特征金字塔自底向上传达强定位特征

### Head

> Head层由三个检测头组成，每个检测头对应不同分辨率的特征图，可以检测不同尺寸的目标

检测头由一个CBL块和一个1*1卷积层构成，每个检测头输出锚框中心的4个坐标偏移量(dx, dy, dw, dh)，1个物体存在的置信度和num_classes个类别概率。

loss函数分为CIOU_Loss和BCE_loss
CIOU_Loss负责衡量预测框与目标框的重叠率、中心点距离、长宽比
BCE_Loss来衡量置信度损失和分类损失
