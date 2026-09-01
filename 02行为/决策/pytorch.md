英伟达官网：
https://developer.nvidia.com/
账号是常用邮箱
密码是：
15853854939

环境配置：
CUDA (Compute Unified Device Architecture) 是NVIDIA推出的并行计算平台和编程模型，它允许开发者利用GPU的强大计算能力进行通用计算。CUDA提供了一整套工具链，包括编译器(nvcc)、驱动、库等，让开发者能够用C/C++/Python调用GPU来执行大规模计算任务

conda用于切换环境，需要具备。


cuda toolkit安装：
http://developer.nvidia.com/cuda-toolkit-archive
CUDA Toolkit 是 NVIDIA 公司提供的一套完整的开发工具包，用于创建和运行利用 NVIDIA GPU 进行加速计算的应用程序。它不仅仅是 CUDA 本身，而是一个包含编译器、库、调试工具和文档的综合性开发环境 。


cudnn安装：
https://developer.nvidia.com/cudnn
最新
https://developer.nvidia.com/cudnn-archive
历史版本
cuDNN (CUDA Deep Neural Network library) 是构建在CUDA之上的深度学习加速库，专门针对神经网络中的常见操作（如卷积、池化、归一化）进行底层优化。它是一个GPU加速的深度神经网络基元库，能够以高度优化的方式实现标准例程


你写的深度学习代码 (PyTorch/TensorFlow)
    ↓
框架底层 (C++)
    ↓
cuDNN (NVIDIA提供的深度学习加速库)
    ↓
CUDA (通用GPU加速平台)
    ↓
GPU硬件
