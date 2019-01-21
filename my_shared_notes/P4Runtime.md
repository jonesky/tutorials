P4Runtime.md

P4 编译器会在编译的时候自动生成控制面的API ，所以， 不需要重启 控制层，当数据层面
部署完成后，控制面 可以直接继续下命令，并且直接支持新的命令（API提供的）

![image](https://ws1.sinaimg.cn/large/005JrW9Kgy1fzebvmxxfxj30u90h00u8.jpg)


P4Runtime 使用了 ProtoBuf  和 gRPC, 工作流如下：

![image](https://ws2.sinaimg.cn/large/005JrW9Kgy1fzeceepyvwj30dz0enjs1.jpg)