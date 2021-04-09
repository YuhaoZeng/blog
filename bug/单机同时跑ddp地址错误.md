# 单机同时跑ddp地址错误问题

同一太服务器进行ddp时，可能会报错RuntimeError: Address already in use
这是因为调用了同一个端口，需要指定一个未被使用的端口，如：
```shell
python -m torch.distributed.launch --master_port xxx
```
参考链接：https://blog.csdn.net/j___t/article/details/107774289
