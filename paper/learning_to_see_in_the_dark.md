# Learning to see in the dark

CVPR 2018

论文地址：https://arxiv.org/pdf/1805.01934.pdf

项目地址：https://cchen156.github.io/SID.html

## abstract

针对低光照和低动态范围实现的网络，raw data作为输入，可以替代大部分pipeline，也提出了新的dataset。



## See-in-the-Dark Dataset

5094张图片，pair是低曝光度图片和高曝光度图片，多个低曝光图像可能匹配同一张高曝光度图像（5094-424）。采用了Sony和Fuji两种相机（对应两种sensor）。



## Pipeline

提到了两种之前的方法L3和Burst。

网络灵感来源于FCN，主要原因是该方法可以表示很多图像处理任务。

Bayer图像输入后拆分为4个通道，因子2下采样，每个通道表示一个颜色信息。同时提取原始图像的black level并利用Amplification Ratio进行scale，再输入FCN网络，网络输出为12通道，最后通过sub-pixel层。恢复为原始图像。作者也用CAN和U-Net进行尝试，默认的网络结构是U-Net。作者还提到残差连接没有什么效果（这一点存疑，因为BayerUnify（NTIRE 2019）的论文中确实采用了residual connection）。Amplification Ratio贴近相机预设ISO，该因数人为设定，可以影响网络图像的亮度。

训练时采用L1 loss，Adam优化。输入为raw图，label为长曝光图（sRGB色域，libraw生成）。随机裁剪为512\*512的图片，学习率10\*-4  。



![1588599001503](C:\Users\dell\AppData\Roaming\Typora\typora-user-images\1588599001503.png)



## experiment

对比了传统的pipeline和denoising&burst。评价指标用了PSNR和SSIM，采用了经过白平衡的reference图像，主要目的是消除color bias的影响。还采用了Mechanical Turk platform的主观评测。结果上都是论文的号，这里不贴结果了。

控制变量实验探究了网络结构，输入图像和loss等因素的影响。U-net网络结构更优，raw data效率更好（这一点存疑，raw data不一定比sRGB效果差，可能在这一个任务上确实是这样），L1 loss训练结果更好，packing比masked效果更好（packing是指把图像拆分为低分辨率和多通道，也是论文的默认方法，masking待补充，参考论文[3]），采用了直方图均衡化结果会显著下降（也就是网络模型不能很好地学习到直方图均衡化的性质）。

![1588602915306](C:\Users\dell\AppData\Roaming\Typora\typora-user-images\1588602915306.png)

## discussion

没有做HDR tone mapping（光照对比度不好），SID的数据集不含人像和运动物体。

网络的增益系数选取很严格（用于AutoISO），且没有做跨sensor的泛化性能测试。

运行时间有待提高（0.38s&0.66s for Sony&Fuji）。



## Reference

1. H. Jiang, Q. Tian, J. E. Farrell, and B. A. Wandell. Learning
   the image processing pipeline. IEEE Transactions on Image
   Processing, 26(10), 2017. 4
2. S.W. Hasinoff, D. Sharlet, R. Geiss, A. Adams, J. T. Barron,
   F. Kainz, J. Chen, and M. Levoy. Burst photography for high
   dynamic range and low-light imaging on mobile cameras.
   ACM Transactions on Graphics, 35(6), 2016. 1, 2, 4, 5

3. M. Gharbi, G. Chaurasia, S. Paris, and F. Durand. Deep joint
   demosaicking and denoising. ACM Transactions on Graphics,
   35(6), 2016. 2, 7

