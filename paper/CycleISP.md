# CycleISP

## 1.introduction

针对去噪领域数据集难以生成、人工合成数据集训练后网络泛化性能差这一问题进行研究，提出了CycleISP用于raw图像和sRGB图像的noise和clean图片转换，并训练了网络实现SOTA的去噪效果。

作者提到，传统的噪声往往是sRGB域上的加性高斯白噪声，但是实际噪声是RAW域上的，经过去马赛克和后续ISP流程会导致噪声变为空间颜色相关，且影响分布，导致网络泛化能力弱。

作者提出的网络主要将噪声加在RAW域上，且方法是设备无关的，来贴近真实网络。



## 2.network structure

网络大体分为两个部分，RGB2RAW和RAW2RGB，还有额外的颜色注意力模块和noise注入模块。颜色模块主要是在RAW2RGB中加入额外的色彩通道修正，noise注入只在生成噪声图中用到，训练中用不到。

训练分为两步：两个模块分别独立训练，再联合起来进行fine-tune，网络结构如下图所示。

![image-20200828202234369](https://raw.githubusercontent.com/YuhaoZeng/blog/master/pic/20200828202238.png)



### 2.1 RGB2RAW

RGB2RAW采用的是多个RRG模块（RRG模块详细说明见2.3节），不需要外部相机参数，卷积层输出是3通道图片，最后过一个mosaic层得到Raw图像，loss上选用了L1 loss和L1 loss的log之和。log部分可以使得网络平等处理图片值，避免网络过分关注高光区域。

![image-20200828160312676](https://raw.githubusercontent.com/YuhaoZeng/blog/master/pic/20200828201422.png)



### 2.2 RAW2RGB

相较于前一个网络有两个个不同的地方，一是Raw图像的处理，先进行pack raw拆分为4通道，图像尺寸大小减半，再最后用一个PixelShuffle还原到原始图像的3通道大小。二是color correction，这一部分是用来对RAW图像做颜色校正的，这一部分先让RGB图片通过一个有很大高斯核（size为12）的高斯模糊，确保图片中保留了颜色信息而忽略结构语义信息，然后通过两个RRG模块进行特征学习再输出。

网络训练采用的是L1 loss。



### 2.3 RRG：Recursive Residual Group

RRG是核心模块之一，这部分的思想是用注意力机制和残差连接来模拟low-level的图像处理环节。网络同时使用了空间注意力机制（SA）和通道注意力机制（CA），这部分和通用的注意力机制差不多，SA的就对像素做pooling，再过卷积层得到权重，CA的就对每个通道做pooling，再过卷积层得到权重，不过这里只用了AveragePooling，不知道为什么没有加入MaxPooling。

![image-20200828163845271](https://raw.githubusercontent.com/YuhaoZeng/blog/master/pic/20200828201440.png)





### 2.4 Joint Fine-Tuning

这部分是在独立训练网络后再进行微调，采用的损失函数如下图所致，带上角标的表示是网络生成的图片，这里RAW2RGB网络只接收第二个项对应的loss的梯度，而RGB2RAW网络接收这两项的loss的梯度，作者认为这可以更好的提升最后的图片效果。

![image-20200828170754808](https://raw.githubusercontent.com/YuhaoZeng/blog/master/pic/20200828201639.png)



## 3 Noise Data Generation
Raw图像的生成是利用RGB2RAW网络将干净的RGB图片转换为干净的Raw图片，打开noise injection模块，噪声参数的选择参考了这篇文章[Tim Brooks, Ben Mildenhall, Tianfan Xue, Jiawen Chen,Dillon Sharlet, and Jonathan T Barron. Unprocessing images for learned raw denoising. In CVPR, 2019.](https://arxiv.org/abs/1811.11127)，这样就从任意的RGB图片生成了RAW的pair。

RGB图像的生成采用noisyRaw作为输入，经过RAW2RGB网络输出noisyRGB图像，得到RGBpair。

作者还采用SIDD数据集进行了进一步的训练，训练的方式如下图所示，让网络更贴近真实噪声。





![image-20200828171216248](https://raw.githubusercontent.com/YuhaoZeng/blog/master/pic/20200828201647.png)



## 4 Denoising Network

文章设计了对Raw和RGB的去噪网络，网络结构一样，只在输入输出数据处理上有差异。网络结构

![image-20200828173854244](https://raw.githubusercontent.com/YuhaoZeng/blog/master/pic/20200828201657.png)



## 5 Result
结果上大部分没什么指的注意的，无非是超过了现有方法，感兴趣的可以去论文原文看。
作者提到了泛化性能和几种设计的影响，结果表明泛化性能还是比较不错的，有一定的迁移性。作者还考虑了残差连接，颜色校正，CA，SA和CASA的排布的影响，结果表明残差连接和颜色校正对网络影响比较大，注意力机制有影响大不是特别关键，从排布方式来讲也是parallel最好，但是影响其实都不是很大，残差连接对性能的提升也不太需要证明，颜色校正可能是由于进一步提高了对颜色的学习能力，从而使得网络色彩分布更好，注意力机制则是在基础上进一步提升效果。

![image-20200828174244602](https://raw.githubusercontent.com/YuhaoZeng/blog/master/pic/20200828201713.png)