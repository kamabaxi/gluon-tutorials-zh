# 使用卷积神经网络的物体检测

当我们讨论对图片进行预测时，到目前为止我们都是谈论分类。我们问过这个数字是0到9之间的哪一个，这个图片是鞋子还是衬衫，或者下面这张图片里面是猫还是狗。

![](../img/dog2.jpg)

但现实图片会更加复杂，它们可能不仅仅包含一个主体物体。**物体检测**就是针对这类问题提出，它不仅是要分析图片里有什么，而且需要识别它在什么位置。我们使用在[机器学习简介](../chapter_crashcourse/introduction.md)那章讨论过的图片作为样例，并对它标上主要物体和位置。

![](../img/catdog_label.svg)

可以看出物体检测跟图片分类有几个不同点：

1. 图片分类器通常只需要输出对图片中的主物体的分类。但物体检测必须能够识别多个物体，即使有些物体可能在图片中不是占主要版面。严格上来说，这个任务一般叫**多类物体检测**，但绝大部分研究都是针对多类的设置，所以我们这里为了简单去掉了”多类“
1. 图片分类器只需要输出将图片物体识别成某类的概率，但物体检测不仅需要输出识别概率，还需要识别物体在图片中的位置。这个通常是一个括住这个物体的方框,通常也被称之为**边界框**（bounding box）。

但也看到物体检测跟图片分类有类似之处，都是对一块图片区域判断其包含的主要物体。因此可以想象我们在前面介绍的基于卷积神经网络的图片分类可以被应用到这里。

这一章我们将介绍数个基于卷积神经网络的物体检测算法的思想。

## R-CNN：区域卷积神经网络

这是基于卷积神经网络的物体检测的奠基之作。其核心思想是在对每张图片选取多个区域，然后每个区域作为一个样本进入一个卷积神经网络来抽取特征，最后使用分类器来对齐分类，和一个回归器来得到准确的边框。

![R-CNN](../img/rcnn.svg)

具体来说，这个算法有如下几个步骤：

1. 对每张输入图片使用一个基于规则的“选择性搜索”算法来选取多个提议区域
1. 跟[微调迁移学习](./fine-tuning.md)里那样，选取一个预先训练好的卷积神经网络并去掉最后一个输入层。每个区域被调整成这个网络要求的输入大小并计算输出。这个输出将作为这个区域的特征。
1. 使用这些区域特征来训练多个SVM来做物体识别，每个SVM预测一个区域是不是包含某个物体
1. 使用这些区域特征来训练线性回归器将提议区域

直观上R-CNN很好理解，但问题是它可能特别慢。一张图片我们可能选出上千个区域，导致一张图片需要做上千次预测。虽然跟微调不一样，这里训练可以不用更新用来抽特征的卷积神经网络，从而我们可以事先算好每个区域的特征并保存。但对于预测，我们无法避免这个。从而导致R-CNN很难实际中被使用。

## Fast R-CNN：快速的区域卷积神经网络

Fast R-CNN对R-CNN主要做了两点改进来提升性能。


1. 考虑到R-CNN里面的大量区域可能是相互覆盖，每次重新抽取特征过于浪费。因此Fast R-CNN先对输入图片抽取特征，然后再选取区域
1. 代替R-CNN使用多个SVM来做分类，Fast R-CNN使用单个多类逻辑回归，这也是前面教程里默认使用的。

![Fast R-CNN](../img/fast-rcnn.svg)

从示意图可以看到，使用选择性搜索选取的区域是作用在卷积神经网络提取的特征上。这样我们只需要对原始的输入图片做一次特征提取即可，如此节省了大量重复计算。

Fast R-CNN提出兴趣区域池化层（Region of Interest (RoI) pooling），它的输入为特征和一系列的区域，对每个区域它将其均匀划分成$n \times m$的小区域，并对每个小区域做最大池化，从而得到一个$n\times m$的输出。因此不管输入区域的大小，RoI池化层都将其池化成固定大小输出。

下面我们仔细看一下RoI池化层是如何工作的，假设对于一张图片我们提出了一个$4\times 4$的特征，并且通道数为1.

```{.python .input  n=7}
from mxnet import nd

x = nd.arange(16).reshape((1,1,4,4))
x
```

然后我们创建两个区域，每个区域由一个长为5的向量表示。第一个元素是其对应的物体的标号，之后分别是`x_min`，`y_min`，`x_max`，和`y_max`。这里我们生成了$3\times 3$和$4\times 3$大小的两个区域。

RoI池化层的输出大小是`num_regions x num_channels x n x m`。它可以当做一个样本个数是`num_regions`的普通批量进入到其他层进行训练。

```{.python .input  n=10}
rois = nd.array([[0,0,0,2,2], [0,0,1,3,3]])
nd.ROIPooling(x, rois, pooled_size=(2,2), spatial_scale=1)
```

## Faster R-CNN：更快速的区域卷积神经网络

Fast R-CNN沿用了R-CNN的选择性搜索方法来选择区域。这个通常很慢。Faster R-CNN做的主要改进是提出了**区域提议网络**（region proposal network, RPN）来替代选择性搜索。它是这么工作的：

1. 在输入特征上放置一个$填充为1通道是256的3\times 3$卷积。这样每个像素，连同它的周围8个像素，都没映射成一个长为256的向量。
1. 以对每个像素为中心生成数个大小和长宽比预先设计好的$k$个默认边框，通常也叫**锚框**。
1. 对每个边框，使用其中心像素对应的256维向量作为特征，RPN训练一个2类分类器来判断这个区域是不是含有任何感兴趣的物体还是只是背景，和一个4维输出的回归器来预测一个更准确的边框。
1. 对于所有的锚框，个数为$nmk$如果输入大小是$n\times m$，选出被判断成还有物体的，然后前他们对应的回归器预测的边框作为输入放进接下来的RoI池化层

![Faster R-CNN](../img/faster-rcnn.svg)

虽然看上有些复杂，但RPN思想非常直观。首先提议预先配置好的一些区域，然后通过神经网络来判断这些区域是不是感兴趣的，如果是，那么再预测一个更加准备的边框。这样我们能有效降低搜索任何形状的边框的代价。


**吐槽和讨论欢迎点**[这里](https://discuss.gluon.ai/t/topic/2510)
