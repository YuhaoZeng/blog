# Pytorch DataParallel and DistributedDataParallel

最近试着使用Pytorch跑单机多卡训练，遇到了不少问题，做个总结和教程方便未来观看。我自己也是一个新手，很多东西总结的不好，有问题请多多指教，**不懂的地方可以看参考文档，很多东西写的比我详细**（本文只针对单机多卡训练，多机多卡训练未经过验证，请酌情观看）
环境：
python 3.7
pytorch 1.4.0

### 2021.3.30更新
知乎有一篇不错的教程和分析，可以参考
[[原创][深度][PyTorch] DDP系列第一篇：入门教程](https://zhuanlan.zhihu.com/p/178402798)
## DataParallel

DataParallel是官方最早提供的一个库，使用非常简单，一行就够了

```python
import torch.nn as nn

device = torch.device("cuda")
model = model.to(device)
model = nn.DataParallel(model)
```



DataParallel的缺点也很明显，只能在单台计算机上使用，而且速度很慢（相较于其他几种加速方式来说），最大的问题在于DataParallel采用了数据分布，但是loss是在某一块卡上计算的，这必然导致会出现比较严重的负载不均衡问题，其他卡用了很少的显存而主卡的显存使用很多，batch size稍微大一点主卡的显存就会炸了，因此更推荐使用其他的方式进行并行计算。



## DistributedDataParallel

DistributedDataParallel（DDP）也是官方提供的一个并行计算库，相比于前者，该方式可以更好地进行多机多卡运算，更好的进行并行计算，负载均衡，运行效率也更高，支持的库和方法也更多，虽然使用起来较为麻烦，但对于追求性能来讲是一个更好的选择。

DDP会自动将库分配给n个进程并在n个gpu上运行，loss的计算也是在各个gpu上进行的，很大程度上解决了分配不均衡的问题。

### 模型设置
- 调用init设置通信后端
用于指定通信后端，一般设置nccl就可以，这是nvidia的GPU的参数设定，更具体的设定请参考官方文档。
```python
torch.distributed.init_process_group(backend='nccl')
```

- 添加local_rank参数
local rank是为了DDP运行的时候进行节点进程制定，这个参数设定进去就可以，我们在自己进行计算时不需要人为给定参数，调用启动器的时候会自动设置
```python
parser = argparse.ArgumentParser()
parser.add_argument('--local_rank', default=-1, type=int,
                    help='node rank for distributed training')
args = parser.parse_args()
```

- 对dataloader设置sampler
数据传递的时候需要将数据分发给不同的GPU，需要在dataloader里指定sampler进行分发，注意制定了sampler的话shuffle就没用了，不要赋值
```python
train_sampler = torch.utils.data.distributed.DistributedSampler(train_dataset)

train_loader = torch.utils.data.DataLoader(train_dataset, ...your args..., sampler=train_sampler)
```

- 模型包装
最后使用模型把数据加载到GPU中，进行反向传播
```python
model = Yourmodel(...)
model.cuda()
model = torch.nn.parallel.DistributedDataParallel(model, device_ids=[args.local_rank], output_device=[args.local_rank])
```

其余部分的使用和正常的网络训练过程是一样的，正常设置输入输出，optimizer，计算loss。loss部分会自动根据GPU进行分发计算，正常调用loss.backward()即可，不需要额外指定loss的parallel。

### 模型保存加载
这部分挺坑的，之前在网上找了一些代码，但是在自己的电脑上用的时候都用问题，这里最后用的是官方的一份代码，需要用到cpu进行一些转载，但是没什么bug

- 模型保存
模型的保存和正常的有点区别，在用DDP（DataParallel也是一样）进行封装后，原有的模型已经被封装起来了，在运行的时候进行分发，模型保存在```model.module```模块，一种比较方便的办法是保存这部分原始模型，这里列举了只保存参数的方法，保存网络的方法，也是同理的，去掉state_dict()即可
```python
model_without_ddp = model.module
checkpoint = torch.save(model_without_ddp.state_dict(),PATH)
```

- 模型加载

    - 多卡加载
      多卡加载一般用于模型训练中断了以后重训练的方式，需要使用map_location这一参数指定网络模型的加载情况，一些教程推荐使用cuda的方式加载，但我在实验的时候总会出现进程上的错误，这里我采用了官方的一种cpu加载方式。
      这里的加载方式是保存时保存原始模型，加载时也加载原始模型，通过修改DDP模型中的module模块使得网络的模型被赋值，免去了在GPU上加载的问题。
    ```python
    model = Yourmodel(...)
    model.cuda()
    model = nn.parallel.DistributedDataParallel(model, device_ids=[args.local_rank], output_device=[args.local_rank])
    # save checkpoint
    model_without_ddp = model.module
    checkpoint = torch.save({'model_state_dict':model_without_ddp.state_dict()},PATH)
    
    #load checkpoint with multiple GPUs
    if args.resume == True:
        resume_file = torch.load('checkpoint.pth.tar', map_location="cpu")
        model_without_ddp.load_state_dict(resume_file['model_state_dict'])
    ```
    - 单卡加载
      单卡加载一般用于进行测试的时候只需要用一块卡测试，这时候我们就需要制定用某一块卡进行数据加载了，其实理论上是差不多的，只是修改一点参数
    ```python
    model = Yourmodel(...)
    model.cuda()
    model = nn.parallel.DistributedDataParallel(model, device_ids=[args.local_rank], output_device=[args.local_rank])
    
    # save checkpoint
    model_without_ddp = model.module
    checkpoint = torch.save({'model_state_dict':model_without_ddp.state_dict()},PATH)
    
    #load checkpoint with single GPU
    if args.test == True:
        model_test = Yourmodel(...) #notice that your model don't need parallel here
        model_test.eval()
        model_test.cuda()
        resume_file = torch.load('checkpoint.pth.tar', map_location="cuda:0")
        model_test.load_state_dict(resume_file['model_state_dict'])
    ```
    

### 代码运行
代码运行过程中，需要调用DDP对应的启动器，nproc_per_node一般设置为你用于训练的节点数量即可，有4块卡就设置为4
```python
python -m torch.distributed.launch --nproc_per_node=YOUR_GPU_NUM yourscript.py
```
## 参考文档
1.[pytorch官方的一个教程](https://pytorch.org/tutorials/intermediate/ddp_tutorial.html)以及其部分[中文翻译](https://blog.csdn.net/qq_36387683/article/details/107589883)
2.[本文的部分行文逻辑来源及其他几种数据并行方法的说明](https://zhuanlan.zhihu.com/p/98535650)
3.[pytorch官方的segmentation文档，内部使用了DDP(直接搜索args.resume)](https://github.com/pytorch/vision/blob/master/references/segmentation/train.py)
4.[并行原理解释，也有代码，介绍的很详细](https://www.cnblogs.com/yh-blog/p/12877922.html)
5.[简明教程，里面也有官方的一些链接，可以参考](https://zhuanlan.zhihu.com/p/113694038)
6.[介绍了一些其他的部分，可以适当参考](https://blog.csdn.net/m0_38008956/article/details/86559432)
