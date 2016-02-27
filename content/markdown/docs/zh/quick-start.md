# 快速开始指南

---

## SINGA 启动

SINGA安装指南请参考 [安装](installation.html) 页面

### 启动 Zookeeper

SINGA 使用 [zookeeper](https://zookeeper.apache.org/) 协调训练过程。在SINGA运行前请确认zookeeper服务已经开启。

如果你的zookeeper是使用我们thirdparty文件夹中的脚本安装的，你可以使用如下命令启动zookeeper服务：

    #goto top level folder
    cd  SINGA_ROOT
    ./bin/zk-service.sh start

(`./bin/zk-service.sh stop` 停止 zookeeper 服务).

否则，你要自己启动一个zookeeper，但不要使用默认的端口，请这样修改`conf/singa.conf`:

    zookeeper_host: "localhost:YOUR_PORT"

## 单机模式运行

运行单机模式的SINGA与使用集群管理软件如 [Mesos](http://mesos.apache.org/) 或者 [YARN](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html) 运行SINGA正好相反。

### 单节点训练

单节点训练时，本地会启动一个进程运行SINGA，可以参考SINGA中在[CIFAR-10](http://www.cs.toronto.edu/~kriz/cifar.html) 上训练 [CNN 模型](http://papers.nips.cc/paper/4824-imagenet-classification-with-deep-convolutional-neural-networks) 的例子。
超参数的设置遵循[cuda-convnet](https://code.google.com/p/cuda-convnet/)，更多细节请参考 [CNN example](cnn.html)。


#### 数据准备和作业配置

下载数据集，为训练和测试创建数据碎片（data shard）：

    cd examples/cifar10/
    cp Makefile.example Makefile
    make download
    make create

训练数据集和测试数据集分别会创建在 *cifar10-train-shard* 和 *cifar10-test-shard* 文件夹下，同时和产生一个 *image_mean.bin* 文件，包含了所有图片特征的均值。

训练这个CNN模型的代码都是由SINGA的内建层提供的，不需要再写任何代码，用户只需执行脚本 (*../../bin/singa-run.sh*) 为SINGA提供作业配置文件 (*job.conf*) 即可。基于SINGA编程，请参考[编程指南](programming-guide.html).

#### 非并行训练

默认的集群拓扑结构仅有一个工作者和一个服务器，也就是说，训练数据集和神经网络都没有划分。

使用下面的命令启动训练：

    # goto top level folder
    cd ../../
    ./bin/singa-run.sh -conf examples/cifar10/job.conf


可以使用下面的命令列出当前正在执行的作业：

    ./bin/singa-console.sh list

    JOB ID    |NUM PROCS
    ----------|-----------
    24        |1

下述命令用于杀死作业：

    ./bin/singa-console.sh kill JOB_ID


日志和作业信息会生成在 */tmp/singa-log* 文件夹中，可以修改 *conf/singa.conf* 中的`log-dir` 变量来改变日志和作业信息存储位置。


#### 异步并行训练

    # job.conf
    ...
    cluster {
      nworker_groups: 2
      nworkers_per_procs: 2
      workspace: "examples/cifar10/"
    }

SINGA 中的 [异步训练](architecture.html) 可以通过启动多工作者组来启用。比如，我们可以按上述修改 *job.conf*，这样就有了2个工作者组。默认每个工作者组有一个工作者。因为每个进程被设定包含2个共作者，这两个工作者将会在同一个进程中运行。因此，他们会运行内存中的[Downpour](frameworks.html) 训练框架。用户不必显式地为每个工作者（组）分割数据，只需要给每个工作者（组）设置一个随机的数据集偏移量，工作者会像在不同的数据划分上运行一样。

    # job.conf
    ...
    neuralnet {
      layer {
        ...
        sharddata_conf {
          random_skip: 5000
        }
      }
      ...
    }

运行命令是：

    ./bin/singa-run.sh -conf examples/cifar10/job.conf

#### 同步并行训练

    # job.conf
    ...
    cluster {
      nworkers_per_group: 2
      nworkers_per_procs: 2
      workspace: "examples/cifar10/"
    }

SINGA中的 [同步训练](architecture.html) 可以通过发起一个工作者组中多个工作者来启用。如上所示，我们修改 *job.conf* 得到一个工作者组中有两个工作者。因为都在同一个工作者组，这些工作者会同步地运行，这个框架是内存的 [sandblaster](frameworks.html)框架。
模型会被划分给两个工作者，具体地说，每个层（layer）会被切片，切片后的layer仅有 `B/g`个特征实例，其中`B`是 mini-batch中的实例个数， `g` 是工作者组中的工作者个数。也可以用 [other schemes](neural-net.html) 来划分层或者神经网络，所有其他设置跟不划分的设置一样。

    ./bin/singa-run.sh -conf examples/cifar10/job.conf

### 在集群中训练

我们可以通过修改集群配置将以上两个训练框架扩展到集群中：

    nworker_per_procs: 1

这样，每个进程只创建一个工作者线程，这会导致工作者只能在不同的进程（节点）中创建。*SINGA_ROOT/conf/* 下的 *hostfile* 必须指明集群中的节点，比如：

    logbase-a01
    logbase-a02

zookeeper 的位置必须正确配置，比如：

    #conf/singa.conf
    zookeeper_host: "logbase-a01"

运行命令和单节点训练的命令相同：

    ./bin/singa-run.sh -conf examples/cifar10/job.conf

## 使用Mesos运行

请参考[Mesos上的分布式训练](mesos.html)。

## 下一步

[编程指南](programming-guide.html) 会介绍如何给SINGA提交一个训练作业。
