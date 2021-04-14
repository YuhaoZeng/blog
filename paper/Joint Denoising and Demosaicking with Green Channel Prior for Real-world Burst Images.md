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
