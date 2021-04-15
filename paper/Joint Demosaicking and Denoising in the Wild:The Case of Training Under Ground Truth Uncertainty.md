# Joint Demosaicking and Denoising in the Wild:The Case of Training Under Ground Truth Uncertainty

AAAI，2021

论文地址：http://arxiv.org/abs/2101.04442

## Abstract

针对joint demosaicking and denoising（JDD）任务中gt常常是不完美的这一问题，设计了一套联合分布来模拟不确定性，并设计了ELBO loss来训练网络和估计模糊参数。是一篇切入角度很新奇的文章。

## Introduction & related work

核心思想没变，主要是说明了data driven类训练方法并没有考虑到gt图像的问题，而作者提出的方法可以更加有效的评估gt图像，提高泛化性能，同时又用ELBO loss控制网络不要收敛到局部最小值。

## Wild-JDD Methodology

一般来讲，我们获得的数据集pair是一张退化图片和一张真实图片的估计。作者把重建过程分为了两部分，（1）创建一个两阶段退化模型来估算所有退化参数，（2）推导了数据相似性的表达式来进行优化。同时作者还提出了fine-tune的方式来拟合位于分布外的输入。

### two-stage data degradation

