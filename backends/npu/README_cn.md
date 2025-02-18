# 飞桨自定义接入硬件后端(昇腾NPU)

简体中文 | [English](./README.md)

请参考以下步骤进行硬件后端(昇腾NPU)的编译安装与验证

## 昇腾NPU系统要求

| 芯片类型  | CANN版本     |
| --------- | -------- |
| 芯片类型 | 昇腾910B |
| CANN版本 | [CANN 8.0.RC1](https://support.huawei.com/enterprise/zh/ascend-computing/cann-pid-251168373/software) |
| 驱动版本 | [23.0.3](https://support.huawei.com/enterprise/zh/ascend-computing/ascend-hdk-pid-252764743/software) |

**注意**：昇腾910A芯片的支持请切换到 [release/2.6](https://github.com/PaddlePaddle/PaddleCustomDevice/blob/release/2.6/backends/npu/README_cn.md) 分支进行编译安装。

## 环境准备与源码同步

```bash
# 1) 拉取镜像，注意此镜像仅为开发环境，镜像中不包含预编译的飞桨安装包
#    此镜像的构建脚本与 dockerfile 位于 tools/dockerfile 目录下
# 昇腾910B芯片 - 系统环境下查看 lspci | grep d802 是否有输出
registry.baidubce.com/device/paddle-npu:cann80RC1-ubuntu20-x86_64-gcc84-py310
registry.baidubce.com/device/paddle-npu:cann80RC1-ubuntu20-aarch64-gcc84-py310

# 2) 参考如下命令启动容器，ASCEND_RT_VISIBLE_DEVICES 指定可见的 NPU 卡号
docker run -it --name paddle-npu-dev -v $(pwd):/work \
    --privileged --network=host --shm-size=128G -w=/work \
    -v /usr/local/Ascend/driver:/usr/local/Ascend/driver \
    -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi \
    -v /usr/local/dcmi:/usr/local/dcmi \
    -e ASCEND_RT_VISIBLE_DEVICES="0,1,2,3,4,5,6,7" \
    registry.baidubce.com/device/paddle-npu:cann80RC1-ubuntu20-$(uname -m)-gcc84-py310 /bin/bash

# 3) 克隆 PaddleCustomDevice 源码
git clone https://github.com/PaddlePaddle/PaddleCustomDevice
cd PaddleCustomDevice
```

## PaddlePaddle 安装与运行

### 源码编译安装

```bash
# 1) 进入硬件后端(昇腾NPU)目录
cd backends/npu

# 2) 编译之前需要先保证环境下装有飞桨安装包，直接安装飞桨 CPU 版本即可
# 默认开发镜像中不含有飞桨安装包，可通过如下地址安装 PaddlePaddle develop 分支的 nightly build 版本的安装包
pip install paddlepaddle -i https://www.paddlepaddle.org.cn/packages/nightly/cpu/

# 3) 编译选项，是否打开单元测试编译，默认值为 ON
export WITH_TESTING=OFF

# 4) 执行编译脚本 - submodule 在编译时会按需下载
bash tools/compile.sh

# 5) 编译产出在 build/dist 路径下，使用 pip 安装
pip install build/dist/paddle_custom_npu*.whl
```

### 基础功能检查

```bash
# 1. 列出可用硬件后端
python -c "import paddle; print(paddle.device.get_all_custom_device_type())"
# 预期得到如下输出结果
['npu']

# 2. 检查当前安装版本
python -c "import paddle_custom_device; paddle_custom_device.npu.version()"
# 预期得到如下输出结果
version: 0.0.0
commit: 9bfc65a7f11072699d0c5af160cf7597720531ea
cann: 8.0.RC1

# 3. 飞桨健康检查
python -c "import paddle; paddle.utils.run_check()"
# 预期得到如下输出结果
Running verify PaddlePaddle program ...
PaddlePaddle works well on 1 npu.
PaddlePaddle works well on 8 npus.
PaddlePaddle is installed successfully! Let's start deep learning with PaddlePaddle now.
```

### 模型训练和推理示例

```bash
# 运行简单训练和推理示例
python tests/test_LeNet_MNIST.py

# 预期得到如下输出结果 - 训练输出
Epoch [1/2], Iter [01/14], reader_cost: 0.89279 s, batch_cost: 42.20599 s, ips: 97.04784 samples/s, eta: 0:19:41
Epoch [1/2], Iter [02/14], reader_cost: 0.44657 s, batch_cost: 21.10753 s, ips: 194.05393 samples/s, eta: 0:09:29
... ...
Epoch [2/2], Iter [14/14], reader_cost: 0.07168 s, batch_cost: 0.08018 s, ips: 51086.10163 samples/s, eta: 0:00:00
Epoch ID: 2, Epoch time: 1.36502 s, reader_cost: 1.00354 s, batch_cost: 1.12250 s, avg ips: 42009.72047 samples/s
Eval - Epoch ID: 2, Top1 accurary:: 0.81091, Top5 accurary:: 0.99036

# 预期得到如下输出结果 - 推理输出
I0103 19:11:34.570551 43520 program_interpreter.cc:214] New Executor is Running.
I0103 19:11:34.589533 43520 analysis_predictor.cc:1684] CustomDevice is enabled
... ...
I0103 19:11:34.590348 43520 ir_params_sync_among_devices_pass.cc:144] Sync params from CPU to npu:0
--- Running analysis [adjust_cudnn_workspace_size_pass]
--- Running analysis [inference_op_replace_pass]
--- Running analysis [ir_graph_to_program_pass]
I0103 19:11:34.592526 43520 analysis_predictor.cc:1867] ======= optimize end =======
I0103 19:11:34.592568 43520 naive_executor.cc:200] ---  skip [feed], feed -> inputs
I0103 19:11:34.592684 43520 naive_executor.cc:200] ---  skip [save_infer_model/scale_0.tmp_0], fetch -> fetch
Output data size is 10
Output data shape is (1, 10)
```

### 环境变量

| 主题   | 变量名称                         | 类型   | 说明                              | 默认值                                                       |
| -------- | -------------------------------- | ------ | --------------------------------- | ------------------------------------------------------------ |
| 调试     | CUSTOM_DEVICE_BLACK_LIST  | String   | 在黑名单内的算子会异构到CPU上运行 | "" |
| 调试     | FLAGS_npu_check_nan_inf | Bool   | 是否开启所有NPU算子输入输出检查   | False |
| 调试     | FLAGS_npu_blocking_run | Bool   | 是否开启强制同步执行所有 NPU 算子 | False |
| 性能分析 | FLAGS_npu_profiling_dir | String | 设置 Profiling 数据保存目录       | "ascend_profiling" |
| 性能分析 | FLAGS_npu_profiling_dtypes | Uint64 | 指定需要采集的 Profiling 数据类型 | 见 [runtime.cc](https://github.com/PaddlePaddle/PaddleCustomDevice/blob/develop/backends/npu/runtime/runtime.cc#L31) |
| 性能分析 | FLAGS_npu_profiling_metrics | Uint64 | 设置 AI Core 性能指标采集项       | 见 [runtime.cc](https://github.com/PaddlePaddle/PaddleCustomDevice/blob/develop/backends/npu/runtime/runtime.cc#L36) |
| 性能加速 | FLAGS_npu_storage_format  | Bool   | 支持 Conv/BN 等算子的昇腾私有化格式 | False |
| 算子编译 | FLAGS_npu_jit_compile  | Bool   | 是否开启算子在线编译 | True |
