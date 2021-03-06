# **在 OpenPAI 上运行 Experiment**

NNI 支持在 [OpenPAI](https://github.com/Microsoft/pai) （简称 pai）上运行 Experiment，即 pai 模式。 在使用 NNI 的 pai 模式前, 需要有 [OpenPAI](https://github.com/Microsoft/pai) 群集的账户。 如果没有 OpenPAI 账户，参考[这里](https://github.com/Microsoft/pai#how-to-deploy)来进行部署。 在 pai 模式中，会在 Docker 创建的容器中运行 Trial 程序。

## 设置环境

参考[指南](QuickStart.md)安装 NNI。

## 运行 Experiment

以 `examples/trials/mnist-annotation` 为例。 NNI 的 YAML 配置文件如下：

```yaml
authorName: your_name
experimentName: auto_mnist
# 并发运行的 Trial 数量
trialConcurrency: 2
# Experiment 的最长持续运行时间
maxExecDuration: 3h
# 空表示一直运行
maxTrialNum: 100
# 可选项: local, remote, pai
trainingServicePlatform: pai
# 可选项: true, false  
useAnnotation: true
tuner:
  builtinTunerName: TPE
  classArgs:
    optimize_mode: maximize
trial:
  command: python3 mnist.py
  codeDir: ~/nni/examples/trials/mnist-annotation
  gpuNum: 0
  cpuNum: 1
  memoryMB: 8196
  image: openpai/pai.example.tensorflow
  dataDir: hdfs://10.1.1.1:9000/nni
  outputDir: hdfs://10.1.1.1:9000/nni
# 配置访问的 OpenPAI 集群
paiConfig:
  userName: your_pai_nni_user
  passWord: your_pai_password
  host: 10.1.1.1
```

注意：如果用 pai 模式运行，需要在 YAML 文件中设置 `trainingServicePlatform: pai`。

与本机模式，以及[远程计算机模式](RemoteMachineMode.md)相比，pai 模式的 Trial 有额外的配置：

* cpuNum 
    * 必填。 Trial 程序的 CPU 需求，必须为正数。
* memoryMB 
    * 必填。 Trial 程序的内存需求，必须为正数。
* image 
    * 必填。 在 pai 模式中，Trial 程序由 OpenPAI 在 [Docker 容器](https://www.docker.com/)中安排运行。 此键用来指定 Trial 程序的容器使用的 Docker 映像。
    * [Docker Hub](https://hub.docker.com/) 上有预制的 NNI Docker 映像 [nnimsra/nni](https://hub.docker.com/r/msranni/nni/)。 它包含了用来启动 NNI Experiment 所依赖的所有 Python 包，Node 模块和 JavaScript。 生成此 Docker 映像的文件在[这里](https://github.com/Microsoft/nni/tree/master/deployment/Dockerfile.build.base)。 可以直接使用此映像，或参考它来生成自己的映像。
* dataDir 
    * 可选。 指定了 Trial 用于下载数据的 HDFS 数据目录。 格式应为 hdfs://{your HDFS host}:9000/{数据目录}
* outputDir 
    * 可选。 指定了 Trial 的 HDFS 输出目录。 Trial 在完成（成功或失败）后，Trial 的 stdout， stderr 会被 NNI 自动复制到此目录中。 格式应为 hdfs://{your HDFS host}:9000/{输出目录}

完成并保存 NNI Experiment 配置文件后（例如可保存为：exp_pai.yml），运行以下命令：

    nnictl create --config exp_pai.yml
    

来在 pai 模式下启动 Experiment。 NNI 会为每个 Trial 创建 OpenPAI 作业，作业名称的格式为 `nni_exp_{experiment_id}_trial_{trial_id}`。 可以在 OpenPAI 集群的网站中看到 NNI 创建的作业，例如： ![](./img/nni_pai_joblist.jpg)

注意：pai 模式下，NNIManager 会启动 RESTful 服务，监听端口为 NNI 网页服务器的端口加1。 例如，如果网页端口为`8080`，那么 RESTful 服务器会监听在 `8081`端口，来接收运行在 Kubernetes 中的 Trial 作业的指标。 因此，需要在防火墙中启用端口 `8081` 的 TCP 协议，以允许传入流量。

当一个 Trial 作业完成后，可以在 NNI 网页的概述页面（如：http://localhost:8080/oview）中查看 Trial 的信息。

在 Trial 列表页面中展开 Trial 信息，点击如下的 logPath： ![](./img/nni_webui_joblist.jpg)

接着将会打开 HDFS 的 WEB 界面，并浏览到 Trial 的输出文件： ![](./img/nni_trial_hdfs_output.jpg)

在输出目录中可以看到三个文件：stderr, stdout, 以及 trial.log

如果希望将 Trial 的模型数据等其它输出保存到HDFS中，可在 Trial 代码中使用 `NNI_OUTPUT_DIR` 来自己保存输出文件，NNI SDK会从 Trial 的容器中将 `NNI_OUTPUT_DIR` 中的文件复制到 HDFS 中。

如果在使用 pai 模式时遇到任何问题，请到 [NNI Github](https://github.com/Microsoft/nni) 中创建问题。