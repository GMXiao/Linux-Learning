## Tensorflow Pytorch 研究

### Pytorch简介
由Facebook发布的深度学习框架，简洁、易用、支持动态计算图。

深度学习框架通常侧重于可用性或速度， 而PyTorch表明这两个目标实际上是兼容的，它提供了命令式的和Python式的编程风格，易于调试。并且与其他流行的科学计算库一致，保持了高效性且支持硬件加速器，如GPU。

PyTorch的成功源于将平衡了速度和易用性的设计。整体架构的设计基于四个主要原则：

    Pythonic：数据科学家熟悉Python语言，其编程模型和工具，PyTorch应该是该生态系统的一流成员。
    易用性：PyTorch致力于使编写模型，数据加载器和优化器尽可能简单和高效。机器学习固有的复杂性应在内部由PyTorch库处理，并隐藏在直观的API之后。
    提供稳定的性能：在以易用性为原则的同时，应该提供稳定的性能表现。
    可扩展性：通过保持Pytorch内部实现的简单性，可节省时间来实现其他的附加功能。

## 整体架构设计：
    从Python解释器高效地运行深度学习算法是一个极富挑战性的挑战：例如，全局解释器锁限制了python程序的并发性。
    高效的C++内核：
        尽管PyTorch与Python生态系统紧密集成，但大多数PyTorch为了高性能都是用C++编写的。核心库libtorch 实现了张量数据结构，GPU和CPU操作以及基本的并行原语。它还提供了自动微分系统，包括大多数内置函数的梯度公式。由于C++作为内核，使用的PyTorch操作组成的运算可在多线程中执行，且不需要持有Python全局解释器锁。
    分离控制和数据流：
        PyTorch在其控制流（即程序分支，循环）和数据流（即张量及其上执行的操作）之间保持严格的分离。控制流的解析由运行在CPU上的Python和 C++ 代码处理，并且所有的操作调用呈线性顺序。
        Pytorch利用CUDA流机制可以在GPU上异步执行操作，系统因此可以同时执行CPU上的Python代码与GPU上的张量运算以提高整体的性能。
    多进程方案：
        由于具有全局解释器锁（GIL），Python的默认实现不允许并发线程并行执行。为了解决这个问题，Python社区建立了一个标准的 “multiprocessing” 模块，其中包含许多通用的程序，这些程序使用户可以轻松生成子进程而且实现了基本的进程间通信原语。但是，该模块在处理大型数组时效率很低，因此，PyTorch将Python multiprocessing 模块扩展为 torch.multiprocessing，该模块会自动将发送到其他进程的张量的数据移动到共享内存，而不是通过进程间通道来发送。
        Pytorch的这种设计极大地提高了性能，并使进程间的隔离更弱，使得编程模型更类似于常规线程。用户可以轻松实现可在独立GPU上运行的高度并行的程序。
    自定义张量缓存分配器：
        几乎每个运算都必须动态分配一个输出张量以保存其执行结果，因此动态内存分配器的速度至关重要。 PyTorch可以依靠一些优化的库在CPU上处理此任务。但是，在GPU上，例行程序“cudaFree”可能会阻塞该程序的调用，直到GPU上所有队列先前的工作完成。为避免此问题，PyTorch实现了一个自定义的分配器，该分配器建立了CUDA内存的高速缓存，无需使用CUDA原有的API。
        为了进一步提高其有效性，此分配器针对深度学习的特定内存使用模式进行了调整。例如，它会将分配512字节的倍数，以避免出现碎片问题。此外，它为每个CUDA流（工作队列）维护一个独特的内存池。

    使用引用计数来进行内存管理：
        用户设计的模型在训练时会利用所有可用的内存。因此，为了提供出色的性能，PyTorch将内存视为需要谨慎管理的稀缺资源。其他框架使用垃圾回收来管理张量内存，运行在其上的模型会定期检查系统状态，枚举已用的张量对象并释放其他的内存。但是这种方法利用周期性地检查事实上推迟了内存的释放，导致了程序总体上会使用更多的内存，而且这种开销是不可接受的。
        PyTorch采用了不同的方法：它依靠引用计数的方案来跟踪每个张量的使用次数，一旦该计数达到零立即释放底层内存。PyTorch通过集成Python的引用计数机制，既可以跟踪 libtorch 库的内部引用，也可以跟踪用户在其 Python 代码中做出的外部引用，这样可确保在不需要张量时准确释放内存。但需要注意的是，这种方法只能在已经使用引用计数的语言中使用（如CPython，Swift，但PyPy或Lua之类的语言不能使用）。




## Pytorch在Mobile中的发展现状
现阶段很多技术需要在边缘设备上进行计算来减少时延、保护用户隐私、提升用户交互体验。Pytorch Mobile 适用于iOS和Android系统上部署的端到端的工作流程。目前Pytorch Mobile已有测试版本可供使用，官方表明它的稳定版本将很快推出。当前，官方已经给出 Android 和 iOS 版本的“快速入门”指南，详情可见：https://pytorch.org/mobile/home

Pytorch Mobile 的关键特点：

    适用于iOS，Android
    提供的API涵盖了将ML集成到移动应用程序中所需的常见预处理和集成任务
    通过TorchScript IR支持跟踪和脚本编写
    支持用于Arm CPU的XNNPACK浮点内核库
    QNNPACK集成用于8位量化内核。包括对每通道量化，动态量化等的支持
    <!-- 构建级别的优化和选择性编译取决于用户应用程序所需的运算符，即应用程序的最终二进制大小取决于应用程序所需的实际运算符 -->
    即将支持GPU，DSP，NPU等硬件后端2

OpenCL
异构系统并行编程的开源标准
OpenCL 3.0规范于2020年9月30日发布

OpenCL通过将计算量很大的代码转移到高速处理器或高速设备上来加速应用程序。OpenCL开发人员使用基于C或C++内核语言对程序进行编码，以在高速设备上并行执行。