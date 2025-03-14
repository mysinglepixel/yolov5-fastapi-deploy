# yolov5-fastapi-deploy
Record problems encountered during deployment and their solutions

## 所谓API部署

在非生产环境中，无需考虑种种限制，只需在有合适算力的电脑上配置好环境即可本地部署。但在实际工程化的生产环境中，跨平台调用、高并发和系统解耦的需求要求使用API或者其他非本地部署方法。

在模型本身训练好后，将模型文件转为中间表示，再使用推理引擎进行剪枝和量化，即可封装为API进行使用了

## YOLOv5

