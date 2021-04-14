# Joint Denoising and Demosaicking with Green Channel Prior for Real-world Burst Images

论文连接：http://arxiv.org/abs/2101.09870

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

### D.Intra-Fram Module

借鉴了(这篇文章)[https://arxiv.org/abs/1903.07291]
