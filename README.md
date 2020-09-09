如何在Kubernetes集群中实现Hyperledger Fabric外部链码
==================================================

**更新**本文已更新到最新的LTS版本（2.2.x），包含最新工件的Github存储库也是如此。

最近发布的Hyperledger Fabric 2.0受到了欢迎，因为它引入了一些有趣的功能来管理链码，其新的分散式链码治理，私有数据增强功能和使用外部链码启动器的功能也得到了管理。

最后提到的最后一个功能消除了docker对启动链码的依赖性，这在某些环境中导致设置区块链网络时遇到许多困难，特别是在不使用[Hyperledger Fabric官方指南]上的**docker-compose**设置时。文档]（https://hyperledger-fabric.readthedocs.io/en/release-2.0/test_network.html）。

由于区块链组件的分发是通过Docker完成的，因此已经使用Kubernetes作为容器编排平台开发了许多用例，以启动整个或部分区块链网络，并且Docker守护程序在同级中的依赖性导致了一些在生产环境中可能不可接受的配置，例如使用对等方工作节点的Docker Socket来部署链码容器，从而使这些容器脱离Kubernetes域，或者使用Docker-in-Docker（ DinD）解决方案，要求这些容器具有特权。

由于这些限制和限制，新发行的版本提供了如何部署这些链码的首选方法，在本教程中，是与[Laura Esbri](https://medium.com/u/2516c80fb4e4?source=post_page-----fd01d7544523----------------------)，我们将了解如何使用“外部链码启动器”部署它们。

* * *

先决条件
=============

以下是为了使一个简单的Fabric网络在dev Kubernetes集群中工作所需的清单。

* **Kubernetes集群**。使用简单的工具，例如[**minikube**]（https://kubernetes.io/docs/setup/learning-environment/minikube/）或单节点[**kubeadm**]，它完全可以是一个集群。 （https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/）集群。后者是我们将在此处展示的环境。您将需要工具来管理Kubernetes集群，例如**kubectl**。设置K8s集群不在本文讨论范围之内。
* **Hyperledger Fabric 2.2.0 docker镜像**。当我们启动Kubernetes yaml描述符的部署时，将拉出映像。
* **超级账本Fabric 2.2.0二进制文件**。我们将需要它们来创建crypto-config和通道伪像。您可以在此[link](https://github.com/hyperledger/fabric/releases/download/v2.0.1/hyperledger-fabric-linux-amd64-2.0.1.tar.gz)中下载它们。
*您将在此[Github存储库]中找到我们将要使用的代码（https://github.com/vanitas92/fabric-external-chaincodes）。

* * *

**安装二进制文件**
========================

使用以下说明安装所需的二进制文件：

```
wget [https://github.com/hyperledger/fabric/releases/download/v2.2.0/hyperledger-fabric-linux-amd64-2.2.0.tar.gz](https://github.com/hyperledger/fabric/releases/download/v2.2.0/hyperledger-fabric-linux-amd64-2.2.0.tar.gz)tar -xzf hyperledger-fabric-linux-amd64-2.2.0.tar.gz\# Move to the bin path  
mv bin/\* /bin\# Check that you have successfully installed the tools by executing  
configtxgen --version\# Should print the following output:  
\# configtxgen:  
\#  Version: 2.2.0  
\#  Commit SHA: 5ea85bc54  
\#  Go version: go1.14.4  
\#  OS/Arch: linux/amd64
```

启动网络
=====================

一旦我们建立了Kubernetes集群并准备就绪，我们将启动网络。但是首先，我们需要生成必要的基本加密材料，以建立网络的身份和起源块。为了使新的链码生命周期正常工作，需要在`configtx.yaml`文件中进行一些更改，这些更改是使用外部链码启动器所必需的。

区块链网络将由一个具有3个实例的RAFT订购服务，两个组织（每个组织都有一个对等实体（_org1_和_org2_））以及每个组织的CA组成。它已经被编码在`configtx.yaml`和`crypto-config.yaml`中，并且不需要修改这些文件。

在configtx.yaml上有一些新的配置选项，作为新的生命周期和认可策略。可以在此文件中定义这些选项，以设置在网络中发生背书操作时允许哪个角色签名。在这种情况下，我们将其设置为属于组织MSP的对等方，并带有`Endorsement`策略选项：

具有认可政策的组织定义

此后，我们可以设置应用程序功能以在发生认可时设置签名策略。有两个规则要定义，“ LifecycleEndorsement”和“ Endorsement”规则。我们将它们设置为“重要程度”规则，该规则表明，任何背书都必须获得一半以上的网络参与者的批准（需要50％+1签名）：

具有认可政策规则的应用程序功能

一旦对网络配置进行了设置，就可以生成加密材料和区块链网络的创世块。我们将通过以下命令使用“fabricOps.sh”脚本：

> **注意**：确保此脚本具有执行权限。更改为744渗透率就足够了。

```
$ ./fabricOps.sh start
```

这将创建所有加密材料，例如对等订购者和CA的所有证书，以及通道的创始块。

我们首先需要为Hyperledger Fabric工作负载创建一个新的名称空间，并使用以下命令创建一个名称空间：

```
$ kubectl create ns hyperledger
```

创建一个文件夹以存储所有Hyperledger Fabric工作负载的持久性数据：

```
$ mkdir /home/storage
```

创建命名空间后，就可以部署工作负载了：

> **注意**：检查并修改订购者部署上附加的卷的主机路径，YAML使用绝对路径，此处假定存储库位于“ _ / home_”文件夹中。

```
$ kubectl create -f orderer-service/
```

通过发出以下命令来检查一切是否正常：

```
$ kubectl get pods -n hyperledger\### Should print a similar output  
NAME                        READY   STATUS    RESTARTS   AGE  
orderer0-58666b6bd7-pflf7    1/1     Running   0          5m47s  
orderer1-c4fd65c7d-c27ll    1/1     Running   0          5m47s  
orderer2-557cb7865-wlcmh    1/1     Running   0          5m47s
```

现在创建org1工作负载，它将部署该组织的对等方和CA：

> **注2 **：同样，再次检查并修改对等体，CA和CLI部署上附加的卷的主机路径，Yaml使用绝对路径，此处假定存储库位于`_ / home_中。 `文件夹。


```
$ kubectl create -f org1/
```

通过发出以下命令来检查一切是否正常：

```
$ kubectl get pods -n hyperledger\### Should print a similar output  
NAME                          READY   STATUS    RESTARTS   AGE  
ca-org1-84945b8c7b-9px4s      1/1     Running   0          19m  
cli-org1-bc9f895f6-zmmdc      1/1     Running   0          2m56s  
orderer0-58666b6bd7-pflf7      1/1     Running   0          79m  
orderer1-c4fd65c7d-c27ll      1/1     Running   0          79m  
orderer2-557cb7865-wlcmh      1/1     Running   0          79m  
peer0-org1-798b974467-vv4zz   1/1     Running   0          19m
```

对**org2**工作负载重复相同的步骤：

> **注3**：同样，再次检查和修改对等体，CA和CLI部署上附加的卷的主机路径，Yaml使用绝对路径，此处假定存储库位于`/home`中。 夹。

```
$ kubectl create -f org2/
```

通过发出以下命令来检查一切是否正常：

```
$ kubectl get pods -n hyperledger\### Should print a similar output  
NAME                          READY   STATUS    RESTARTS   AGE  
ca-org1-84945b8c7b-9px4s      1/1     Running   0          71m  
ca-org2-7454f69c48-q8lft      1/1     Running   0          2m20s  
cli-org1-bc9f895f6-zmmdc      1/1     Running   0          55m  
cli-org2-7779cc8788-8q4ns     1/1     Running   0          2m20s  
orderer0-58666b6bd7-pflf7      1/1     Running   0          131m  
orderer1-c4fd65c7d-c27ll      1/1     Running   0          131m  
orderer2-557cb7865-wlcmh      1/1     Running   0          131m  
peer0-org1-798b974467-vv4zz   1/1     Running   0          71m  
peer0-org2-5849c55fcd-mbn5h   1/1     Running   0          2m19s
```

以上所有工作负载都应立即显示在_hyperledger_名称空间中。

> **注4**：如果存在CrashloopBackoff错误，请检查路径是否正确设置以及加密文件是否在位并正确设置。 通过发出“ kubectl logs pod_name -n hyperledger”命令来检查日志，指向有故障的pod。

* * *

设置网络渠道
===============================

部署所有工作负载后，我们准备创建通道并将对等方加入到该通道中。 输入**org1**的cli pod：

```
$ kubectl exec -it cli\_org1\_pod\_name sh -n hyperledger
```

进入cli pod后，执行以下命令：

```
$ peer channel create -o orderer0:7050 -c mychannel -f ./scripts/channel-artifacts/channel.tx --tls true --cafile $ORDERER\_CA\### Should print a similar output  
2020-03-06 11:54:57.582 UTC \[channelCmd\] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized  
2020-03-06 11:54:58.903 UTC \[cli.common\] readBlock -> INFO 002 Received block: 0
```

频道“mychannel”已创建并可以使用。 接下来，将**org1**的对等方加入频道：

```
$ peer channel join -b mychannel.block\### Should print a similar output  
2020-03-06 12:01:41.608 UTC \[channelCmd\] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized  
2020-03-06 12:01:41.688 UTC \[channelCmd\] executeJoin -> INFO 002 Successfully submitted proposal to join channel
```

我们将在** org2 ** cli上执行相同的步骤，但是由于该通道已经由** org1 **创建，因此我们将从订单服务中获取创世块。 首先输入广告连播：

```
$ kubectl exec -it cli\_org2\_pod\_name sh -n hyperledger
```

进入cli pod后，执行以下命令：

```
$ peer channel fetch 0 mychannel.block -c mychannel -o orderer0:7050 --tls --cafile $ORDERER\_CA\### Should print a similar output  
2020-03-06 12:18:14.880 UTC \[channelCmd\] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized  
2020-03-06 12:18:14.895 UTC \[cli.common\] readBlock -> INFO 002 Received block: 0
```

然后从创世块加入频道：

```
$ peer channel join -b mychannel.block\### Should print a similar output  
2020-03-06 12:20:41.475 UTC \[channelCmd\] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized  
2020-03-06 12:20:41.561 UTC \[channelCmd\] executeJoin -> INFO 002 Successfully submitted proposal to join channel
```

您可以通过执行以下命令随时检查是否有任何对等方加入了该通道：

```
$ peer channel list\### Should print a similar output  
2020-03-06 12:22:41.102 UTC \[channelCmd\] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized  
Channels peers has joined:   
mychannel
```

* * *

安装外部链码
================================

现在我们要做有趣的事情stuff。我们将把大理石链代码部署为外部链代码。您可以在[fabric-sample Github存储库]（https://github.com/hyperledger/fabric-samples/tree/master/chaincode/marbles02/go）中找到原始链码，但是为了将其部署为外部Chaincode ，我们需要对导入和init函数进行一些更改，以将其声明为外部chaincode服务器。

新版本的Hyperledger还带有新的链码生命周期过程，可在您的对等节点上安装链码并在通道上启动它。为了使用[外部链码]（https://hyperledger-fabric.readthedocs.io/en/release-2.0/cc_launcher.html）功能，我们也必须使用此新过程。

我们要做的第一件事是配置对等体以处理外部链码。外部构建器仅基于“ buildpacks”，因此我们创建了3个脚本，“ detect”，“ build”和“ release”。这3个脚本必须在对等容器中定义。这些脚本可以在“ buildpack / bin”文件夹中找到。

> **注意5 **：请确保这3个脚本在buildpack / bin文件夹中具有执行权限。这些脚本以root用户身份在对等容器中执行，因此将权限更改为744应该足够了。否则，将导致externalbuilder执行失败。

可以在builders-config.yaml中的org1文件夹中找到构建器的定义。除了_core \ _chaincode \ _externalbuilders_选项之外，该文件具有所有用于配置core.yaml对等方的默认选项，该选项具有如下所示的自定义生成器配置。

core.yaml中的ExternalBuilder定义

因为对等方的Kubernetes部署描述符具有环境变量来配置选项，所以这些环境变量将覆盖core.yaml中的默认选项。之所以这样做是因为环境变量不接受数组格式，并且这种方式可以使用自己的配置来配置每个对等方，但是可以建立此构建器。该网络中的所有组织都使用它。

新的生命周期过程打包了链代码并以与以前版本不同的方式安装链代码。由于我们正在使用外部链码功能，因此链码代码不必在对等容器本身中进行编译和安装，而不必在另一个容器中。在对等进程中唯一要安装的代码是能够连接到外部chaincode进程所需的信息。

> **注6：**我们将对org1 **对等链代码执行步骤，因此在org1 cli pod上执行命令。对于org2对等方，将重复这些步骤，但稍后会更改链码的某些配置。

为此，我们需要对打包代码进行打包[https://hyperledger-fabric.readthedocs.io/en/release-2.0/cc_service.html#packaging-chaincode），并附带一些要求。必须有一个“ connection.json”文件，其中包含与外部链码服务器的连接信息。这包括地址，TLS证书和拨号超时配置。

对等链到外部链码的连接定义

该文件需要打包到名为`code.tar.gz`的tar文件中。通过执行以下命令来实现。

> **注6：**您可以在链式代码主机路径存储所指向的主机的cli工具窗格内或cli荚外部执行这些命令。最后，在cli pod中需要生成的tar文件来启动命令，因此我建议从cli pod中执行此操作。如果要在cli pod中执行这些命令，请转到以下路径：`/ opt / gopath / src / github.com / marbles / packaging`

```
$ cd chaincode/packaging  
$ tar cfz code.tar.gz connection.json
```

有了code.tar.gz文件后，我们必须再次将其与另一个名为metadata.json的文件重新打包，该文件包含有关要处理的链码类型，链码所在路径以及 我们想要给该链码的标签。

对等链到外部链码的元数据信息

我们使用以下命令打包这两个文件：

```
$ tar cfz marbles-org1.tgz code.tar.gz metadata.json
```

收到tar文件后，就可以使用新的生命周期过程将其安装在对等方中了。

```
$ peer lifecycle chaincode install marbles-org1.tgz\### Should print a similar output  
2020-03-07 14:33:18.120 UTC \[cli.lifecycle.chaincode\] submitInstallProposal -> INFO 001 Installed remotely: response:<status:200 payload:"\\nGdmarbles:e001937433673b11673d660d142c722fc372905db87f88d2448eee42c9c63064\\022\\006degree" >   
2020-03-07 14:33:18.126 UTC \[cli.lifecycle.chaincode\] submitInstallProposal -> INFO 002 Chaincode code package identifier: marbles:e001937433673b11673d660d142c722fc372905db87f88d2448eee42c9c63064
```

复制链码代码包标识符，以备后用。 不过，您始终可以通过执行以下命令来取回它：

```
$ peer lifecycle chaincode queryinstalled\### Should print a similar output  
Installed chaincodes on peer:  
Package ID: marbles:030eec59c7d74fbb4e9fd57bbd50bb904a715ffb9de8fea85b6a6d4b8ca9ea12, Label: marbles
```

现在我们将对**org2**重复上述相同的步骤，但是由于我们希望有一个不同的Pod为**org2**对等节点提供相同的链码，因此我们必须在`connection`中更改address config选项。 json文件。 像这样修改文件地址值：

```
"address": "chaincode-marbles-org2.hyperledger:7052",
```

然后重复与之前相同的步骤，在org2 cli窗格中执行install命令：

```
$ rm -f code.tar.gz  
$ tar cfz code.tar.gz connection.json  
$ tar cfz marbles-org2.tgz code.tar.gz metadata.json  
$ peer lifecycle chaincode install marbles-org2.tgz\### Should print a similar output  
2020-03-07 15:10:15.093 UTC \[cli.lifecycle.chaincode\] submitInstallProposal -> INFO 001 Installed remotely: response:<status:200 payload:"\\nGmarbles:c422c797444e4ee25a92a8eaf97765288a8d68f9c29cedf1e0cd82e4aa2c8a5b\\022\\006degree" >   
2020-03-07 15:10:15.093 UTC \[cli.lifecycle.chaincode\] submitInstallProposal -> INFO 002 Chaincode code package identifier: marbles:c422c797444e4ee25a92a8eaf97765288a8d68f9c29cedf1e0cd82e4aa2c8a5b
```
像以前一样复制链码代码包标识符。当我们将org2的地址更改为org时，它的哈希值应与org1的哈希值不同。

安装chaincode时对等端发生了什么？如果在对等端定义了一些外部构建器或启动器，则在尝试由对等端自身构建Docker容器的经典选项之前，它们将被执行。当我们定义了“ external-builder”时，执行以下脚本顺序。

“检测”脚本仅评估要安装的链码是否具有“ metadata.json”的“类型”键作为“外部”值。如果该脚本失败，则对等方认为该外部构建器不必构建链码，并迭代到其他外部构建器，直到没有更多定义且回退到Docker构建过程为止。

检测externalbuilder的脚本

如果`detect`脚本成功，则将调用`build`脚本。如果我们在对等体内部构建链码，则此脚本应生成构件或二进制文件，但是由于我们希望作为外部服务使用，因此只需将“ connection.json”文件复制到构建输出目录就足够了。

外部链码的构建脚本

最后，一旦“ build”脚本完成，就会调用“ release”脚本。该脚本负责通过将其放置在发布输出目录中来向对等方提供“ connection.json”。因此，如果要执行链代码的某些调用，则对等方现在知道在哪里调用链代码。

外部链码的发布脚本

* * *

构建和部署外部Chaincode
============================================

一旦在对等节点中安装了链码，就可以构建它们并将其部署到我们的Kubernetes集群中。让我们看一下我们需要对链码进行的更改。

由于链码不提供厂商，也不包含当前所需的模块，因此导入已更改，因此需要导入它们并提供模块。

导入外部链码

在构建代码之前，还必须定义`go.mod`文件以建立模块的版本，这一点很重要。需要启用垫片模块中的最新版本才能启用“外部Chaincode”属性。

带有软件包所需版本的Go.mod

另一个更改是Chaincode Server属性，该属性侦听我们要建立的特定地址和端口。因为我们可能想通过使用Kubernetes yaml描述符来更改它们，所以我们将使用** _ os _ **模块来获取地址的值和链码代码包标识符（** CCID **）。通过这样做，我们可以部署相同的代码映像，但根据需要更改参数。

带有服务器实例定义的链码的主要功能

现在，我们准备使用以下`Dockerfile`构建chaincode容器：

链文件的Dockerfile

链码是使用golang高山图像构建的。构建阶段完成后，将二进制文件传输到裸露的高山映像中，以获取较小的安全映像。执行以下命令：

```
$ docker build -t chaincode/marbles:1.0 .
```

如果一切正常，则应该已准备好部署映像。 将链码部署的文件（org1-chaincode-deployment.yaml和org2-chaincode-deployment.yaml）修改为CHAINCODE \ _CCID **变量，分别为安装链码之前获得的值。

之后，部署它们：

```
$ kubectl create -f chaincode/k8s
```

该服务和部署将与其他工作负载部署在相同的k8s命名空间中。

```
$ kubectl get pods -n hyperledgerNAME                                      READY   STATUS    RESTARTS   AGE  
ca-org1-84945b8c7b-tx59g                  1/1     Running   0          19h  
ca-org2-7454f69c48-nfzsq                  1/1     Running   0          19h  
chaincode-marbles-org1-6fc8858855-wdz7z   1/1     Running   0          20m  
chaincode-marbles-org2-77bf56fdfb-6cdfm   1/1     Running   0          14m  
cli-org1-589944999c-cvgbx                 1/1     Running   0          19h  
cli-org2-656cf8dd7c-kcxd7                 1/1     Running   0          19h  
orderer0-5844bd9bcc-6td8c                 1/1     Running   0          46h  
orderer1-75d8df99cd-6vbjl                 1/1     Running   0          46h  
orderer2-795cf7c4c-6lsdd                  1/1     Running   0          46h  
peer0-org1-5bc579d766-kq2qd               1/1     Running   0          19h  
peer0-org2-77f58c87fd-sczp8               1/1     Running   0          19h
```

现在，我们必须批准每个组织的链码。 这是链码生命周期过程的新功能，每个组织都必须同意批准链码的新定义。 我们将批准org1 **的大理石代码定义。 在** org1 ** cli pod中执行此命令，切记更改** CHAINCODE \ _CCID：**

```
$ peer lifecycle chaincode approveformyorg --channelID mychannel --name marbles --version 1.0 --init-required --package-id marbles:e001937433673b11673d660d142c722fc372905db87f88d2448eee42c9c63064 --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER\_CA\### Should print a similar output  
2020-03-08 10:02:46.192 UTC \[chaincodeCmd\] ClientWait -> INFO 001 txid \[4d81ea5fd494e9717a0c860812d2b06bc62e4fc6c4b85fa6c3a916eee2c78e85\] committed with status (VALID)
```

您可以通过执行以下命令来检查整个网络中的批准状态：

```
$ peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name marbles --version 1.0 --init-require  
d --sequence 1 -o -orderer0:7050 --tls --cafile $ORDERER\_CA\### Should print a similar output  
Chaincode definition for chaincode 'marbles', version '1.0', sequence '1' on channel 'mychannel' approval status by org:  
org1MSP: true  
org2MSP: false
```

现在，我们批准** org2 **。 在** org2 ** cli pod中执行此命令，切记更改** CHAINCODE \ _CCID：**

```
$ peer lifecycle chaincode approveformyorg --channelID mychannel --name marbles --version 1.0 --init-required --package-id marbles:25a9f6fe26161d29af928228ca1db0c41892e26e46335c84952336ee26d1fd93 --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER\_CA\### Should print a similar output  
2020-03-08 10:26:43.992 UTC \[chaincodeCmd\] ClientWait -> INFO 001 txid \[74a89f3c93c10f14c626bd4d6cb654b37889908c9e6f7b983d2cad79f1e82267\] committed with status (VALID)
```

再次检查链码的提交准备情况：

```
$ peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name marbles --version 1.0 --init-required --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER\_CA\### Should print a similar output  
Chaincode definition for chaincode 'marbles', version '1.0', sequence '1' on channel 'mychannel' approval status by org:  
org1MSP: true  
org2MSP: true
```

现在，我们已经获得了所有组织的批准，让我们在渠道中提交此链码的定义。 您可以在任何对等节点上执行此操作：

```
$ peer lifecycle chaincode commit -o orderer0:7050 --channelID mychannel --name marbles --version 1.0 --sequence 1 --init-required --tls true --cafile $ORDERER\_CA --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt\### Should print a similar output  
2020-03-08 14:13:49.516 UTC \[chaincodeCmd\] ClientWait -> INFO 001 txid \[568cb81f821698025bbc61f4c6cd3b4baf1aea632e1e1a8cfdf3ec3902d1c6bd\] committed with status (VALID) at peer0-org1:7051  
2020-03-08 14:13:49.533 UTC \[chaincodeCmd\] ClientWait -> INFO 002 txid \[568cb81f821698025bbc61f4c6cd3b4baf1aea632e1e1a8cfdf3ec3902d1c6bd\] committed with status (VALID) at peer0-org2:7051
```

现在，我们已将链码定义添加到通道，并准备被调用并对其进行查询！ 😄

* * *

测试外部链码
=============================

我们可以测试来自cli pod的调用和查询命令的链码。 这些命令不会被生命周期链代码过程修改，可以在Hyperledger Fabric版本1.x中称为链代码。 首先，让我们在分类帐中创建一些大理石。 在一个cliorg ** org1 **或** org2 **上执行以下命令。

```
$ peer chaincode invoke -o orderer0:7050 --isInit --tls true --cafile $ORDERER\_CA -C mychannel -n marbles --peerAddresses   
peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer  
0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c '{"Args":\["initMarble  
","marble1","blue","35","tom"\]}' --waitForEvent\### Should print a similar output  
2020-03-08 14:23:03.569 UTC \[chaincodeCmd\] ClientWait -> INFO 001 txid \[83aeeaac47cf6302bc139addc4aa38116a40eaff788846d87cc815d2e1318f44\] committed with status (VALID) at peer0-org2:7051  
2020-03-08 14:23:03.575 UTC \[chaincodeCmd\] ClientWait -> INFO 002 txid \[83aeeaac47cf6302bc139addc4aa38116a40eaff788846d87cc815d2e1318f44\] committed with status (VALID) at peer0-org1:7051  
2020-03-08 14:23:03.576 UTC \[chaincodeCmd\] chaincodeInvokeOrQuery -> INFO 003 Chaincode invoke successful. result: status:200
```

这将在分类帐上创建一个_marble1_。 现在创建另一个大理石：

```
$ peer chaincode invoke -o orderer0:7050 --isInit --tls true --cafile $ORDERER\_CA -C mychannel -n marbles --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c '{"Args":\["initMarble","marble2","red","50","tom"\]}' --waitForEvent\### Should print a similar output  
2020-03-08 14:23:40.404 UTC \[chaincodeCmd\] ClientWait -> INFO 001 txid \[8391f9f8ea84887a56f99e4dc4501eaa6696cd7bd6c524e4868bd6cfd5b85e78\] committed with status (VALID) at peer0-org2:7051  
2020-03-08 14:23:40.434 UTC \[chaincodeCmd\] ClientWait -> INFO 002 txid \[8391f9f8ea84887a56f99e4dc4501eaa6696cd7bd6c524e4868bd6cfd5b85e78\] committed with status (VALID) at peer0-org1:7051  
2020-03-08 14:23:40.434 UTC \[chaincodeCmd\] chaincodeInvokeOrQuery -> INFO 003 Chaincode invoke successful. result: status:200
```

从_marble1：_检索信息

```
$ peer chaincode query -C mychannel -n marbles -c '{"Args":\["readMarble","marble1"\]}'{"docType":"marble","name":"marble1","color":"blue","size":35,"owner":"tom"}
```

There are many example commands you can execute with this chaincode, just check the source code of the chaincode for more examples.

You can also check the logs of the chaincode containers by executing the following command:

```
$ kubectl logs chaincode\_pod\_name -n hyperledger\### Should print a similar output  
invoke is running initMarble  
\- start init marble  
\- end init marble  
invoke is running initMarble  
\- start init marble  
\- end init marble  
invoke is running readMarble
```

* * *

奖金轨道！具有外部Chaincodes功能的基于ContractApi的Chaincodes
==========================================================================

作为我们最近一直在玩的新功能，我们现在可以使用2.0版中引入的新ContractApi部署外部链码，这使得开发链码更加容易。我们将部署fabcar链代码，您可以在[这里]找到原始代码（https://github.com/hyperledger/fabric-samples/blob/master/chaincode/fabcar/go/fabcar.go）。修改后的文件将被推送到名为fabcar的文件夹中的仓库中。

我们将需要添加新的shim api，以允许添加允许与对等方进行外部通信的外部Chaincode服务器接口。

go.mod文件也需要导入这两个模块。

这就是使它工作所需的所有修改！安装说明与之相同，只是使用fabcar（而不是大理石）一词。唯一的区别是在approveformyorg，commit和invoke中不需要使用--init-required或--isInit标志。

```
peer lifecycle chaincode approveformyorg --channelID mychannel --name fabcar --version 1.0 --package-id fabcar:005c35f4f172c056723eca09d41e8048e0beaa2712d920c19af837640df7e2aa --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER\_CApeer lifecycle chaincode approveformyorg --channelID mychannel --name fabcar --version 1.0 --package-id fabcar:61ab817a6ad76098d340952e5d8e928d9c5ddff1a53341dc8d0c64b4345564b0 --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER\_CApeer lifecycle chaincode checkcommitreadiness --channelID mychannel --name fabcar --version 1.0 --sequence 1 -o -orderer0:7050 --tls --cafile $ORDERER\_CApeer lifecycle chaincode commit -o orderer0:7050 --channelID mychannel --name fabcar --version 1.0 --sequence 1 --tls true --cafile $ORDERER\_CA --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crtpeer chaincode invoke -o orderer0:7050 --tls true --cafile $ORDERER\_CA -C mychannel -n fabcar --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c '{"Args":\["InitLedger"\]}' --waitForEventpeer chaincode query -C mychannel -n fabcar -c '{"Args":\["QueryAllCars"\]}'
```
您还将在[他们的github存储库]（https://github.com/hyperledger/fabric-samples/tree/master/chaincode/fabcar/external）中找到Hyperledger团队使用外部链码功能的相同链码的原始实现。 ）。它们的实现与我们的实现之间没有多少区别，因此我们假定这是使用外部功能实现此基于契约的链码的正确方法。

未来的工作和改进
===========================

可以进行一些改进，我们很高兴收到社区的反馈，以改进这些外部链码。

* **外部Builders定义为环境变量，而不是configmap **。尽管它以这种方式工作，但我们希望将环境变量中的“外部构建器”定义为其他选项。我们试图将外部构建器定义为环境变量，但是由于预期ExternalBuilders配置为数组，并且环境变量仅接受字符串，因此存在很多错误。 **更新2 **：_你们中的许多人都报告了许多外部构建器定义未成功执行或未正确检测到的问题。这主要是由于对scripts文件夹的权限（从GitHub克隆后未启用执行权限）引起的。为了易于使用，buildpack脚本现在已包含为a_ [_configmap_]（https://github.com/vanitas92/fabric-external-chaincodes/blob/master/org1/builders-config.yaml）并附加在具有更高kubernetes集成的对等Pod._
* **背书策略失败：**由于在通道上提交链码定义时，由于许多背书策略失败，我们不得不禁用了背书和生命周期背书策略。由于这可能需要更好地理解和定义2.0版中的策略，因此我们没有足够的时间正确配置它。 **更新1 **：_ _此问题已得到解决，并且已对新要求进行了更新，以使创始块中编写的代言策略和生命周期代言策略成为可能。
* **手动更改链码的地址配置**。每当我们有一个新的组织时，都需要更改“ connection.json”文件的地址值。如果我们每个组织都有多个背书人，也可以更改。可能会有一些有趣的管道来帮助支持这些类型的链码的部署。

**感谢关注！**
