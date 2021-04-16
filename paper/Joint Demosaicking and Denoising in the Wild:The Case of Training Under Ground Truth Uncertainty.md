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

一般的噪声建模是高斯白噪声（文中公式（2）），但考虑真实情况，我们能得到的图片y_j是理论真实图片z_j的一个估计，这就导致我们直接采用y_j训练不能达到理论最优的结果。作者提出了一个新的模型，即真实图片z_j服从均值为y_j，方差为(sigma_j^2/lamda)的高斯分布（这一点我个人有点奇怪，为什么是z服从这个分布，而不是y服从这个分布，从道理上来讲，应该后者更加贴近真实，但如果这样推导公式，可能无法进一步化简？）。进一步作者认为方差sigma_j^2服从以alpha和beta_j为参数的逆伽马分布（文中公式（5））。通过这两步退化，理论真实图像z_j就转换为高斯-逆伽马联合分布了。

作者认为，lamda反应了y_j相较于理论真实图像z_j的不确定程度，alpha和beta_j可以解释为2alpha观察和2beta的样本平方偏差（这部分我不太理解，应该是统计里的知识），这两个值的取值参考了[这篇文章，NIPS2019](https://arxiv.org/abs/1908.11314)。

总结一下，采样过程可以分为两步，第一步，根据alpha，beta_j，y_j和lambda采样得到了一组z和sigma_j，这是高斯分布的均值和方差。第二步，根据这个均值和方差，采样得到一个特定的x_j，这样就生成了一组x_j和z_j的pair。

### Maximizing ELBO

主要是为了能够训练网络提出了loss，这一过程是通过计算marginal log-likelihood实现的。这一部分的推导比较复杂，详细的建议看论文原文和其主要的参考文章[变分编码器](https://arxiv.org/abs/1312.6114)

值得注意的是作者在这里对ELBO loss进行了分析，在lamda很大时，该loss会收敛为MSE loss，只关注重建情况却忽略了gt图片的不真实性，这也就解释了为什么其他几项正则化是需要的。

在测试过程中需要的图像是网络输出的y^，噪声图则根据分布定义获得。

### Corrupted Input as a Weakly Informative Prior

受到mosaic2mosaic和noise2noise两篇论文的启发，作者打算用来x_j~替换y_j，但是这可能导致网络学习全等变换。因此，作者把x_j变为其邻域内的随机一个点。但在边缘复杂的区域这可能导致很高的数值变化，因此作者根据这个变换后的点是否落在x_j的2sigma_j邻域来判断是否计算loss，如果超过了邻域范围，也就是该点像素值变化很大，那么就不计算loss，反之则进行计算。

## Illustrative Experimental Results

### Network Architecture

网络的目的是以x为输入，z为输出。当然根据公式推导，线性插值的部分被包含在网络里了，z也不是真正的z，实际输出是y，lambda，alpha和beta四个参数，每个参数是和原图尺寸相同的3channel输出。网络结构则是上下采样+GRDB+skip connection的结构，比较简单可以看原文。

### Experiments on Synthetic Datasets

先是在sRGB图片上加噪声，不过加入的噪声不是全局同方差高斯，而是随着空间改变的（这个和上面提到额NIPS2019文章一样）。网络训练用了DIV2K和Flickr2K，其他参数可以看论文。

结果上来讲，网络的提升大概在0.2-1db不等，在高噪声图片上的提升比较明显。

### Experiments on Realistic Raw Data

经典MSR dataset，linear和sRGB域都做了，提升比较有限，0.2db左右。

### Fine-tuning Out-of-distribution Input

作者在不同的噪声模型上做了实验，主要证明了不同模型的泛化性能

### Ablation Study

（1）ELBO vs MSE
主要是证明了ELBO的正则化项会限制网络学到有缺陷的gt图片，从而达到更高的PSNR峰值。
（2）mask
在fine-tune过程中加入mask map可以有效的避免网络造成边缘模糊。

## Summary

作者针对目前的网络中获得的gt不够好的问题，采用了两阶段的退化过程来建模真实的图片，中间参考了不少变分编码器的内容。论文的思路还是很新颖的，为了能够训练也对loss做了特别的设计，不过实验部分看起来不是很足够，在加噪声的环节还是没有完全遵循真实噪声的情况。总体上还是很值得学习的。
