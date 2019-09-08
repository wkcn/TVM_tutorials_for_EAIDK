# 在EAIDK-310上学习TVM (一) - “远程过程调用”例子的详细介绍

## 导入模块

```python
# 导入NumPy模块
import numpy as np

# 导入TVM模块，和TVM的远程过程调用模块rpc, 工具模块util. 在本例子中，使用了工具模块util获得临时文件夹的地址。
import tvm
from tvm import rpc
from tvm.contrib import util
```

```python
# 导入配置模块，用于保存开发板的IP地址与开发板上TVM远程过程调用服务的端口，以及针对开发板的编译参数。官方例子中没有config模块，而是直接填写IP地址和端口等参数。
import config
```

## 声明一个核函数

我们需要声明一个核函数，它的作用是输入一个长度为1024的张量(Tensor)，返回这个张量逐元素加1.0的结果。

```python
# 创建一个常数节点，值为1024, 保存在变量n中
n = tvm.convert(1024)

# 建立一个张量A，它同时也是一个占位符号，可以作为函数的输入，并且要求输入的张量的形状(shape)为(n, )。因为n是值为1024的常数节点，所以这里要求输入的张量的长度为1024。
A = tvm.placeholder((n,), name='A')

# 建立一个新的张量B，它的形状为(n, )，它是通过lambda函数A[i] + 1.0计算得到的，也就是将张量A逐元素加1.0的结果保存到张量B。
B = tvm.compute((n,), lambda i: A[i] + 1.0, name='B')

# 为张量B的算子建立一个调度，保存到变量s中
s = tvm.create_schedule(B.op)
```

## 编译核函数 
声明好核函数后，可以对函数进行编译。当我们需要在本地机器执行核函数时，使用默认编译参数即可。当我们需要在远程机器执行核函数，而且远程机器的架构和本地机器不一致时，需要进行交叉编译。
```python
# local_demo表示是否在本机编译，如果local_demo为True则在本机（通常是x86架构）编译；如果local_demo为False则对远程机器（比如ARM架构的EAIDK-310开发板）进行交叉编译（因为编译代码的机器和远程机器不是同一台机器，并且架构不同）。
local_demo = False

# 如果是本地机器，使用llvm默认编译参数编译，否则使用交叉编译参数编译。在config.py文件中可以看到target_host='llvm -target=aarch64-linux-gnu'。因为目标机器(EAIDK-310)的架构是ARMv8架构，运行64位Linux操作系统。
if local_demo:
    target = 'llvm'
else:
    target = config.target_host

# 设置编译签名(signature), 其中s是前面声明的核函数的输出算子的调度。[A, B]指函数的输入和输出，注意这里**没有规定**前面的变量一定是输入，后面的变量一定是输出。变量是输入还是输出取决于核函数的声明，这里也可以写成[B, A]，此时就是输出在前输入在后了。target指的是编译参数。name设置了编译出的函数的名字。`tvm.build`函数将会返回一个编译了的函数对象。
func = tvm.build(s, [A, B], target=target, name='add_one')
```

## 导出目标文件到本地机器的临时文件夹

将核函数编译后，编译后的函数对象保存在变量`func`中。我们需要将编译出的目标文件保存到本地机器的临时文件夹中，并且上传到远程机器（开发板）上。

```python
# 获取本机的临时文件夹路径
temp = util.tempdir()
# 我们将目标文件命名为`lib.tar`，获取它保存到临时文件夹下的绝对路径
path = temp.relpath('lib.tar')
# 将目标文件导出的临时文件夹，命名为lib.tar
func.export_library(path)
```

## 建立会话
在运行核函数前，我们需要建立会话。会话分为“本地会话”和“远程会话”：“本地会话”用于在本地机器进行调试，“远程会话”用于在远程机器执行核函数。有了会话，同一套代码能很方便地在本地或远程机器上执行。

```python
if local_demo:
    # 对于本地机器，建立本地会话
    remote = rpc.LocalSession()
else:
    # 对于远程机器，建立远程会话，其中host为远程机器的IP地址，port为远程机器的TVM远程过程调用服务使用的端口。
    host = config.host
    port = config.port
    remote = rpc.connect(host, port)
```

## 上传目标文件

建立会话后，通过会话上传目标文件lib.tar，并读取目标文件.
```python
# 上传目标文件lib.tar
remote.upload(path)
# 读取目标文件，得到编译后的核函数
func = remote.load_module('lib.tar')
```

## 执行核函数
通过会话上传目标文件后，可以读取目标文件并执行核函数。
```python
# 设置上下文为会话中的机器的CPU，上下文指的是运行核函数的环境，比如CPU，GPU.
ctx = remote.cpu()
# 使用NumPy创建一个长度为1024，类型与张量A一致的随机数组，并将这个数组转换为TVM中的多维数组，上下文为ctx.
a = tvm.nd.array(np.random.uniform(size=1024).astype(A.dtype), ctx)
# 建立一个全部为0, 长度为1024的TVM多维数组
b = tvm.nd.array(np.zeros(1024, dtype=A.dtype), ctx)
# 调用核函数 
func(a, b)
# 对比结果
np.testing.assert_equal(b.asnumpy(), a.asnumpy() + 1)
```
当我们使用本地机器调试核函数时，数组a和数组b都在本地机器上，直接在本地机器执行核函数得到结果，再通过调用`asnumpy`函数，在本地机器中将TVM的多维数组转为NumPy的多维数组；当我们使用远程机器，也就是开发板运行核函数时，数组a和数组b将在远程机器上创建，并在远程机器上运行，通过调用`asnumpy`函数，将远程机器上的多维数组通过网络通信传给本地机器，在本地机器上转为NumPy多维数组。

## 性能分析
我们可以对核函数的性能进行评估，比如统计函数的运行时间。
```python
# 核函数的time_evaluator将会返回一个评估核函数运行时间的函数，这个时间是实际执行函数的时间，不包括网络通信的时间。测试次数number为10.
time_f = func.time_evaluator(func.entry_name, ctx, number=10)
# 得到实际实行的时间，并计算均值
cost = time_f(a, b).mean
print('%g secs/op' % cost)
```

## 总结

在本篇文章中，我分析了TVM的官方例程“远程过程调用”的代码，分为了多个部分：核函数的声明、核函数的编译、导出目标文件、建立会话、上传目标文件、核函数的执行与性能测试。建议大家重点理解“核函数的声名”部分，它声明了整个核函数的计算过程。
