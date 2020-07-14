# 模型裁剪

为了更好地满足端侧部署场景下，低内存带宽、低功耗、低计算资源占用以及低模型存储等需求，PaddleX通过集成PaddleSlim实现`模型裁剪`，可提升Paddle Lite端侧部署性能。

## 原理介绍

模型裁剪通过裁剪卷积层中Kernel输出通道的大小及其关联层参数大小，来减小模型大小和降低模型计算复杂度，可以加快模型部署后的预测速度，其关联裁剪的原理可参见[PaddleSlim相关文档](https://paddlepaddle.github.io/PaddleSlim/algo/algo.html#id16)。**一般而言，在同等模型精度前提下，数据复杂度越低，模型可以被裁剪的比例就越高**。

## 裁剪方法
PaddleX提供了两种方式:

**1.用户自行计算裁剪配置(推荐)，整体流程包含三个步骤,**  
> **第一步**： 使用数据集训练原始模型  
> **第二步**：利用第一步训练好的模型，在验证数据集上计算模型中各个参数的敏感度，并将敏感度信息存储至本地文件  
> **第三步**：使用数据集训练裁剪模型（与第一步差异在于需要在`train`接口中，将第二步计算得到的敏感信息文件传给接口的`sensitivities_file`参数）  

> 在如上三个步骤中，**相当于模型共需要训练两遍**，分别对应第一步和第三步，但其中第三步训练的是裁剪后的模型，因此训练速度较第一步会更快。  
> 第二步会遍历模型中的部分裁剪参数，分别计算各个参数裁剪后对于模型在验证集上效果的影响，**因此会反复在验证集上评估多次**。  

**2.使用PaddleX内置的裁剪方案**  
> PaddleX内置的模型裁剪方案是**基于标准数据集**上计算得到的参数敏感度信息，由于不同数据集特征分布会有较大差异，所以该方案相较于第1种方案训练得到的模型**精度一般而言会更低**（**且用户自定义数据集与标准数据集特征分布差异越大，导致训练的模型精度会越低**），仅在用户想节省时间的前提下可以参考使用，使用方式只需一步，  

> **一步**： 使用数据集训练裁剪模型，在训练调用`train`接口时，将接口中的`sensitivities_file`参数设置为'DEFAULT'字符串

> 注：各模型内置的裁剪方案分别依据的数据集为： 图像分类——ImageNet数据集、目标检测——PascalVOC数据集、语义分割——CityScape数据集

## 裁剪实验
基于上述两种方案，我们在PaddleX上使用样例数据进行了实验，在Tesla P40上实验指标如下所示:

### 图像分类
实验背景：使用MobileNetV2模型，数据集为蔬菜分类示例数据，裁剪训练代码见[tutorials/compress/classification](https://github.com/PaddlePaddle/PaddleX/tree/develop/tutorials/compress/classification)

| 模型 | 裁剪情况 | 模型大小 | Top1准确率(%) |GPU预测速度 | CPU预测速度 |
| :-----| :--------| :-------- | :---------- |:---------- |:----------|
|MobileNetV2 | 无裁剪（原模型）| 13.0M | 97.50|6.47ms |47.44ms |
|MobileNetV2 | 方案一(eval_metric_loss=0.10) | 2.1M | 99.58 |5.03ms |20.22ms |
|MobileNetV2 | 方案二(eval_metric_loss=0.10) | 6.0M | 99.58 |5.42ms |29.06ms |

### 目标检测
实验背景：使用YOLOv3-MobileNetV1模型，数据集为昆虫检测示例数据，裁剪训练代码见[tutorials/compress/detection](https://github.com/PaddlePaddle/PaddleX/tree/develop/tutorials/compress/detection)

| 模型 | 裁剪情况 | 模型大小 | MAP(%) |GPU预测速度 | CPU预测速度 |
| :-----| :--------| :-------- | :---------- |:---------- | :---------|
|YOLOv3-MobileNetV1 | 无裁剪（原模型）| 139M | 67.57| 14.88ms |976.42ms |
|YOLOv3-MobileNetV1 | 方案一(eval_metric_loss=0.10) | 34M | 75.49 |10.60ms |558.49ms |
|YOLOv3-MobileNetV1 | 方案二(eval_metric_loss=0.05) | 29M | 50.27| 9.43ms |360.46ms |

### 语义分割
实验背景：使用UNet模型，数据集为视盘分割示例数据，裁剪训练代码见[tutorials/compress/segmentation](https://github.com/PaddlePaddle/PaddleX/tree/develop/tutorials/compress/segmentation)

| 模型 | 裁剪情况 | 模型大小 | mIOU(%) |GPU预测速度 | CPU预测速度 |
| :-----| :--------| :-------- | :---------- |:---------- | :---------|
|UNet | 无裁剪（原模型）| 77M | 91.22 |33.28ms |9523.55ms |
|UNet | 方案一(eval_metric_loss=0.10) |26M | 90.37 |21.04ms |3936.20ms |
|UNet | 方案二(eval_metric_loss=0.10) |23M | 91.21 |18.61ms |3447.75ms |
