# Joint Denoising and Demosaicking with Green Channel Prior for Real-world Burst Images

论文连接：http://arxiv.org/abs/2101.09870
**注意**：办公环境所限本文暂时不好配图，有需要请务必搭配论文使用，后期可能会补上

## Abstract

针对目前的joint denoising and demosaicking（JDD）任务只在单张图片进行且采用加性高斯白噪声的问题，作者提出了在真实图片上用burst images研究该任务，称作JDD-B
。根据绿通道采样率更高，图片质量也更高的情况，作者设计了绿通道作为特征提取，帧间融合的指导的网络GCP-Net。

## Introduction

介绍了JDD-S（jdd in single image）的缺陷，高噪声条件下效果很差（手机相机、低光照条件下该情况尤其明显），用高斯白噪声在真实数据集上泛化性能有限。
说明了burst去噪方法的有效性，绿色通道的信噪比更高（SIDD数据集）。

## Related Work

JDD-S的任务和burst image restoration的任务，后者介绍的比较详细，可以看原文。

## Method

### A.问题建模

真实图片一般不只有高斯噪声，在raw域上有高斯泊松噪声，本文建模为高斯泊松噪声并等效替换为异方差分布高斯噪声。

### B.绿通道先验

作者对50张来自SIDD数据集的图片进行计算，发现绿色通道的SNR显著高于R和B通道。这个可能和相机的感光有关，绿光的感光程度强，R和B一般需要进行额外增益，而G通道一般不用（参考WB过程），而且G的采样率更高，用来做先验肯定有一定道理。

### C.网络结构

两个分支和三个模块，两个分支分别是RGB通道和G通道，三个模块分别是图片内信息、图片间信息和融合模块。

### D.Intra-frame Module

借鉴了[Semantic Image Synthesis with Spatially-Adaptive Normalization, CVPR2019](https://arxiv.org/abs/1903.07291)的思想，绿通道提取到的信息以系数gamma与偏置beta与原始特征进行点乘，最后得到的特征图再并行通过一个spatial attention和channel attention，得到最后的特征图。这里借鉴的原始论文是做语义图像生成的，给出的先验也不是绿色通道，而是语义图像，是一种利用额外的输入作为权重先验的normalization，可以理解为原始的BN层中的gamma和beta从一个单独的vector变成了包含空间新的的tensor，但一些low-level任务中，BN层的使用很少，这个方法效果如何有待验证。
还有一个注意点是这里的spatial attention采用的是双卷积层自适应寻找，没有采用pooling操作。

### E.Inter-frame Module

主要是为了对齐建模帧间的时序依赖性，采用ConvLSTM结构、金字塔处理和可变形卷积来学习帧间的offset。同时考虑到LSTM对大运动的限制性，用了上采样的方式扩大对大运动的感知。

### F.Merge Module

简单的上采样加U-Net结构，但是上采样过程中同样加入了绿通道的guidance，U-Net只用了3次下采样。

### G.Loss

两种，linear域和srgb域的Charbonnier penalty function(L1范数的可微的变体)。其中一部分在[这篇文章](https://blog.csdn.net/weixin_42447651/article/details/80809564)里面有所说明，其中关于l1 loss和l2 loss的部分有些参考价值，只用l1和l2 loss对于pixel任务不一定是最好的，可能会导致某些过度的模糊。

## Experiment

burst数据集的获取利用了视频数据集Vimeo-90K和[Unprocessing](https://people.csail.mit.edu/tfxue/papers/cvpr2019_unprocess.pdf)的方式获得，对得到的raw图片加入噪声，采用RGGB的bayer pattern。每次输入5帧图片并去中间帧作为参考帧，噪声的level设置在sigma_s为10^-4~10^-2，sigma_r为10^-3~10^-1.5，这个level设置其实不算很高。

### Ablation Study

消融实验主要对GCP-Net，一阶段VS两阶段，inter-frame模块，帧的数量进行了研究。
（1）GCP-Net
这一部分算是网络的核心创新点之一。文章的Fig.7给出了可视化结果，学到的gamma更倾向于全局的亮度信息，beta则包含更多的边缘信息和噪声信息，从提取到的特征看，加了GCP的有更多的细节边缘，而不加的则倾向于给出全局模糊的去噪结果。这一点比较神奇，因为这和真实图像的复原过程有点相似，gamma负责保证图像亮度信息基本不变，beta则着重提取了图像各个点的noise水平，并对边缘信息很好的保留，在保证图像低频信息的前提下又没有损失很多的高频特征。但问题也比较明显，就是提取到的特征图仍然带有不少的噪音，这对于复原肯定是不利的。
从数值结果来看，用了GCP会带来0.13db的提升，前面提到的自适应的spatial attention带来0.1db提升，用RB来guide会带来0.6db的下降。

（2）one-stage vs two-stage
主要是对JDD问题应该用一个网络实现DD两个操作还是两个网络分别实现denoise和demosaic并串联进行讨论。也是个老问题了，不同的文章有过不同的结果，关于denoise和demosaic的先后顺序，要不要一起做，都有过一些探讨。对于本文的结构来讲，one-stage的网络效果会更好一些，提升大概在0.5~0.8db左右。

（3）Role of inter-frame module
帧间融合的有效性，主要是说明了不同帧信息需要一个offset模块进行配准，也算比较好理解。

（4）selection of frame number
不同帧数量测试，算是超参调节吧，5和7差不多，7帧会有微小提升。

### Results

实验对比其实不太公平，主要是因为这个领域的论文太少了，所以作者拿burst denoise的网络和demosaic的网络拼了个two-stage的网络作比较，关键在于这个demosaic的网络还不算是SOTA的方法（16年的[文章](http://groups.csail.mit.edu/graphics/demosaicnet/)），还有两篇单图的JDD论文，也是很早的文章，21年的论文了还用这几个对比其实是不太行，不过新的几篇也没怎么开源，不好比较也是正常。
作者在两个视频数据集进行了测试，从PSNR值来看提升有将近一个db了，已经很高了，不过还是那个问题，去噪的网络拿来直接训练JDD，还是不太合理。
真实数据集上用手机拍摄的图片进行了测试，不过没有gt也就没有PSNR值了。从视觉效果看，图片的复原效果还不错，细节纹理确实保留的不错，但局部也会有一些偏色的小问题。

### Summary
本文针对JDD-B问题提出了GCP-Net的方法，取得了一定的效果。green channel含有更多信息是采样率和感光情况等多种因素决定的，本文提出的guide方法可以借鉴，能够保留更多的细节。帧间图像融合这一点我没有很了解，具体可以看burst去噪的相关方法和论文。不过实验比较上还有改进空间，unprocess的过程也可以适当优化。
