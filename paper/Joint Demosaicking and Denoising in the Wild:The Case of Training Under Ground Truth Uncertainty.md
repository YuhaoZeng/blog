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
