---
title: Fuzzbench架构
date: 2025/06/29 11:11:12
categories:
- Dev
tags:
- Fuzz
---

Fuzzbench的代码架构解析

<!-- more -->

## Fuzzbench

使用流程：

- 根据教程创建好虚拟环境，并安装好python的包
- 调用make执行相关的debug和test-run
- 填写yaml，然后使用脚本运行实验

```bash
source .venv/bin/activate
PYTHONPATH=. python3 experiment/run_experiment.py \
--experiment-config ./fuzzbench.yaml \
--benchmarks mbedtls_fuzz_dtlsclient lcms_cms_transform_fuzzer libpcap_fuzz_both libxml2_xml \
--experiment-name $BENCHNAME \
--fuzzers libafl aflplusplus
```

## 运行相关的Make命令

- **生成Makefile**：首先根目录的makefile会检查是否docker/generated.mk不存在，或者需要更新，然后使用docker/generate_makefile.py生成makefile，然后再include进来。

## Fuzzer的三大文件

- builder.Dockerfile 负责下载和构建Fuzzer
- fuzzer.py 提供两个函数，分别用于构建和运行fuzzer。
- runner.Dockerfile 负责准备Fuzzer的运行环境

## Fuzzing相关的镜像

- **project-builder镜像**：每个Benchmark提供的基础的project-builder镜像里面负责存储项目的源码，并放好相关的构建脚本`$SRC/build.sh`。
- **Coverage镜像**：有个特殊的fuzzer叫做Coverage，内部其实是一个LLVM的Libfuzzer。它负责测量实验过程中的coverage：实验过程中提取所有的queue里的种子，然后拿这个插桩的binary跑一遍得到coverage信息。
- **builder-intermediate镜像**（Fuzzer的**builder.Dockerfile**）：`fuzzers/xxx/builder.Dockerfile`：每个Fuzzer提供的两个dockerfile之一。它开头是固定的通过参数传入Paren Image。该镜像负责在原有的project-builder镜像中构建好fuzzer自身。然后得到**builder-intermediate镜像**。

```dockerfile
ARG parent_image
FROM $parent_image

# 编译并准备好fuzzer
```

- **builder镜像**：基于builder-intermediate镜像，使用统一的dockerfile（`docker/benchmark-builder/Dockerfile`），开始用Fuzzer构建Target。它安装了fuzzbench需要的python3，用于运行Fuzzer提供的三大文件之一的fuzzer.py。最后提供了一个fuzzer_build脚本，内部主要运行的是`python3 -c "from fuzzers import utils; utils.initialize_env(); from fuzzers.$FUZZER import fuzzer; fuzzer.build()"`，而给fuzzers提供的编译Target的接口`utils.build_benchmark`内部就是用bash执行`$SRC/build.sh` 这个就是最早project-builder镜像放好的构建脚本。

最终编译好的Target的相关文件会被放到`OUT=/out`。同时fuzzer.py也要将Fuzzer自身的binary放过去。

- **intermediate-runner镜像**（Fuzzer的**runner.Dockerfile**）：从最基础的Base镜像，使用runner.Dockerfile构建，给fuzzer准备一些辅助的环境，比如安装一些包，设置一些环境变量。对于简单的fuzzer，这里是可以是空的。由于从Base重新构建，这里面不含任何Target和Fuzzer的东西。

- **runner镜像**：将builder镜像里面构建好的Fuzzer和Target复制到intermediate-runner镜像，然后复制了一些fuzzbench的源码进去，最后设置了entry point为`docker/benchmark-runner/startup-runner.sh`。使用docker的multi-stage build特性，使用`docker/benchmark-runner/Dockerfile`统一构建。

**相关的debug target**

- **builder-debug镜像**：在builder镜像的基础上增加了个`--build-arg debug_builder=1`。
- **debug-builder-xxx Target**：在builder-debug镜像上给shell。
- **debug-xxx Target**：在runner镜像上给一个shell `--entrypoint "/bin/bash"`
- **test-run Target**：在runner镜像基础上设置`SNAPSHOT_PERIOD=10`，`MAX_TOTAL_TIME=20`

## Fuzzbench的镜像

- **dispatcher-image**：之后会将docker挂载进去，因此下载了docker-client，然后安装了llvm-cov等工具。最后复制了启动脚本startup-dispatcher.sh，它调用`src/experiment/dispatcher.py`。
- **worker-image**：

## 启动一次实验

实验的docker数量 = benchmark数量 * Fuzzer数量 * 重复数量 + 1

这里我们称某个（benchmark，fuzzer，trial）是一个pair，对应一个docker。

处理跑实验的fuzzer容器，还有负责总体管理的dispatcher容器。

主要的入口是`experiment/run_experiment.py`

- **构建镜像**根据上面镜像的构建流程，得到最终runner镜像。具体就是使用docker依次构建Fuzzer，使用Fuzzer的Toolchain构建Fuzz Target，最后将构建好的Target放到Fuzzer一起准备运行。
- 然后会将相关的参数转存并启动dispatcher-image，执行dispatcher.py。
    - dispatcher容器内部会启动scheduler_loop_thread
    - dispatcher容器内部会启动measurer进程。
- 然后一边等待实验结束，一边生成临时的report。

**scheduler**：它会不断获取相关的需要运行的某个pair，然后执行启动流程（start_trials）。内部创建一个专门的脚本（render_startup_script_template），启动对应的runner。

**Measurer**：主函数执行measure_manager_inner_loop，负责发送请求给measurers_cpus数量的子进程。子进程接受到请求就拉取queue然后测量coverage。这里有一个DEFAULT_SNAPSHOT_SECONDS=900s，每隔这么久就会尝试去拉取所有的fuzzer的coverage测量一下。

