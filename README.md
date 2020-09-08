# 如何在Kubernetes集群中实现Hyperledger Fabric外部链码

更新：本文已更新到最新的LTS版本（2.2.x），包含最新工件的Github存储库也是如此。
最近发布的Hyperledger Fabric 2.0受到了欢迎，因为它引入了一些有趣的功能来管理链码，其新的分散式链码治理，私有数据增强功能以​​及使用外部链码启动器的功能。
最后提到的最后一个功能消除了docker对启动链码的依赖性，这在某些环境中导致设置区块链网络时遇到许多困难，特别是在不使用Hyperledger Fabric官方文档中用作教程的docker -compose设置时。
由于通过Docker完成区块链组件的分发，因此已经使用Kubernetes作为容器编排平台开发了许多用例，以启动整个或部分区块链网络，并且Docker守护程序在同级中的依赖导致了一些在生产环境中可能不可接受的配置，例如使用对等方工作节点的Docker Socket来部署链码容器，从而使这些容器脱离Kubernetes域，或者使用Docker-in-Docker（ DinD）解决方案，要求这些容器具有特权。
由于这些限制和限制，新发行的版本提供了如何部署这些链码的首选方法，在本教程中，由Laura Esbri联合制作，我们将了解如何使用外部链码启动器部署它们。
先决条件
以下是为了使一个简单的Fabric网络在dev Kubernetes集群中工作所需的清单。
Kubernetes集群。使用minikube或单节点kubeadm集群等简单工具，它可以完美地成为一个集群。后者是我们将在此处展示的环境。您将需要工具来管理Kubernetes集群，例如kubectl。设置K8s集群不在本文讨论范围之内。
Hyperledger Fabric 2.2.0泊坞窗映像。当我们启动Kubernetes yaml描述符的部署时，将拉出图像。
Hyperledger Fabric 2.2.0二进制文件。我们将需要它们来创建crypto-config和通道伪像。您可以在此链接中下载它们。
您将在此Github存储库中找到我们将要使用的代码。
安装二进制文件
使用以下说明安装所需的二进制文件：
wget https://github.com/hyperledger/fabric/releases/download/v2.2.0/hyperledger-fabric-linux-amd64-2.2.0.tar.gz
tar -xzf hyperledger-fabric-linux-amd64-2.2.0.tar.gz
＃移至bin路径
mv bin / * / bin
＃通过执行
configtxgen --version 检查您是否已成功安装工具
＃应该显示以下输出：
＃configtxgen：
＃版本：2.2.0 
＃提交SHA：5ea85bc54 
＃Go版本：go1.14.4 
＃OS / Arch：linux / amd64
启动网络
一旦我们建立了Kubernetes集群并准备就绪，我们将启动网络。但是首先我们需要生成必要的基本加密材料，以建立网络的身份和起源块。configtx.yaml为了使新的链码生命周期正常工作，需要对文件进行一些更改，这些更改是使用外部链码启动器所必需的。
区块链网络将由具有3个实例的RAFT订购服务，2个组织组成，每个组织都有一个对等点（org1和org2），每个组织都有一个CA。这已被编码的configtx.yaml 和crypto-config.yaml 并没有需要修改这些文件。
configtx.yaml作为新的生命周期和认可策略，有一些新的配置选项。可以在此文件中定义这些选项，以设置在网络中发生背书操作时允许哪个角色签名。在这种情况下，我们使用Endorsement策略选项将其设置为属于组织MSP的对等方：

认可政策的组织定义
此后，我们可以设置应用程序功能以在发生认可时设置签名策略。有两个要定义LifecycleEndorsement的Endorsement规则，和规则。我们将其设置为MAJORITY规则，该规则表明任何背书都必须获得一半以上的网络参与者的批准（需要50％+1签名）：

具有认可政策规则的应用程序功能
一旦对网络配置进行了设置，就可以生成加密材料和区块链网络的创世块。我们将通过fabricOps.sh以下命令使用脚本：
注意：确保此脚本具有执行权限。更改为744渗透率就足够了。
$ ./fabricOps.sh开始
这将创建所有加密材料，例如对等订购者和CA的所有证书，以及通道的创始块。
我们首先需要为Hyperledger Fabric工作负载创建一个新的名称空间，并使用以下命令创建一个名称空间：
$ kubectl创建ns hyperledger
创建一个文件夹，用于存储所有Hyperledger Fabric工作负载的持久性数据：
$ mkdir / home /存储
创建命名空间后，就可以部署工作负载了：
注意：如果需要，请检查并修改订购者部署上附加的卷的主机路径，Yaml使用绝对路径，此处假定存储库位于/home文件夹中。
$ kubectl create -f命令服务/
通过发出以下命令来检查一切是否正常：
$ kubectl get pods -n hyperledger
###应该打印类似的输出
名称就绪状态重新启动年龄
orderer0-58666b6bd7-pflf7 1/1正在运行0 5m47s 
orderer1-c4fd65c7d-c27ll 1/1正在运行0 5m47s 
orderer2-557cb7865-wlcmh 1/1正在运行0 5m47s
现在创建org1工作负载，该工作负载将部署该组织的对等方和CA：
注2：同样，再次检查并修改对等体，CA和CLI部署上附加的卷的主机路径，Yaml使用绝对路径，此处假定存储库位于/home文件夹中。
$ kubectl创建-f org1 /
通过发出以下命令来检查一切是否正常：
$ kubectl get pods -n hyperledger
###应该打印类似的输出
名称READY STATUS RESTARTS AGE 
ca-org1-84945b8c7b-9px4s 1/1运行0 19m 
cli-org1-bc9f895f6-zmmdc 1/1运行0 2m56s 
orderer0-58666b6bd7-pflf7 1/1运行0 79m 
orderer1-c4fd65c7d-c27ll 1/1运行0 79m 
orderer2-557cb7865-wlcmh 1/1运行0 79m 
peer0-org1-798b974467-vv4zz 1/1运行0 19m
对org2工作负载重复相同的步骤：
注3：同样，再次检查并修改对等体，CA和CLI部署上附加的卷的主机路径，Yaml使用绝对路径，此处假定存储库位于/home文件夹中。
$ kubectl创建-f org2 /
通过发出以下命令来检查一切是否正常：
$ kubectl get pods -n hyperledger
###应该打印类似的输出
名称就绪状态重新启动年龄
ca-org1-84945b8c7b-9px4s 1/1正在运行0 71m 
ca-org2-7454f69c48-q8lft 1/1正在运行0 2m20s 
cli-org1-bc9f895f6-zmmdc 1/1正在运行0 55m 
cli-org2-7779cc8788-8q4ns 1/1正在运行0 2m20s订单
0-58666b6bd7-pflf7 1/1正在运行
0131m orderer1-c4fd65c7d-c27ll 1/1正在运行0 131m 
orderer2-557cb7865-wlcmh 1/1正在运行0 131m 
peer0- org1-798b974467-vv4zz 1/1运行0 71m 
peer0-org2-5849c55fcd-mbn5h 1/1运行0 2m19s
以上所有工作负载都应立即出现在hyperledger名称空间中。
注4：如果存在CrashloopBackoff错误，请检查路径是否正确设置以及加密文件是否在位并正确设置。发出kubectl logs pod_name -n hyperledger命令，指向有故障的Pod，检查日志。
设置网络渠道
部署所有工作负载后，我们准备创建通道并将对等方加入到该通道中。输入org1的cli pod ：
$ kubectl exec -it cli_org1_pod_name sh -n hyperledger
进入cli pod后，执行以下命令：
$对等通道创建-o orderer0：7050 -c mychannel -f ./scripts/channel-artifacts/channel.tx --tls true --cafile $ ORDERER_CA
###应该打印类似的输出
2020-03-06 11：54：57.582 UTC [channelCmd] InitCmdFactory-> INFO 001已初始化Endorser和
orderer 连接2020-03-06 11：54：58.903 UTC [cli.common] readBlock- >信息002接收到的块：0
通道mychannel已创建并可以使用。接下来，将org1的对等方加入频道：
$对等通道加入-b mychannel.block
###应该打印类似的输出
2020-03-06 12：01：41.608 UTC [channelCmd] InitCmdFactory-> INFO 001初始化Endorser和
Orderer 连接2020-03-06 12：01：41.688 UTC [channelCmd] executeJoin-> INFO 002成功提交了加入频道的提案
我们将在org2 cli 上执行相同的步骤，但是由于该通道已由org1创建，因此我们将从订单服务中获取创世块。首先输入广告连播：
$ kubectl exec -it cli_org2_pod_name sh -n hyperledger
进入cli pod后，执行以下命令：
$对等通道获取0 mychannel.block -c mychannel -o orderer0：7050 --tls --cafile $ ORDERER_CA
###应该打印类似的输出
2020-03-06 12：18：14.880 UTC [channelCmd] InitCmdFactory-> INFO 001初始化Endorser和
Orderer 连接2020-03-06 12：18：14.895 UTC [cli.common] readBlock- >信息002接收到的块：0
然后从创世块加入频道：
$对等通道加入-b mychannel.block
###应该打印类似的输出
2020-03-06 12：20：41.475 UTC [channelCmd] InitCmdFactory-> INFO 001初始化Endorser和
Orderer 连接2020-03-06 12：20：41.561 UTC [channelCmd] executeJoin-> INFO 002成功提交了加入频道的提案
您可以通过执行以下命令随时检查是否有任何对等方加入了该通道：
$对等频道列表
###应该打印类似的输出
2020-03-06 12：22：41.102 UTC [channelCmd] InitCmdFactory-> INFO 001 
已初始化Channels对等方的Endorser和
Orderer 连接：mychannel
安装外部链码
现在我们要做有趣的事情stuff。我们将把大理石链代码部署为外部链代码。您可以在fabric-sample Github存储库中找到原始链码，但是为了将其部署为外部链码，我们需要对导入和init函数进行一些更改，以将其声明为外部链码服务器。
新版本的Hyperledger还带有新的链码生命周期流程，可在您的对等节点上安装链码并在通道上启动它。为了使用外部链码功能，我们还必须使用此新过程。
我们要做的第一件事是配置对等体以处理外部链码。外部构建器仅基于buildpacks，因此我们创建了3个脚本detect，build和release。这3个脚本必须在对等容器中定义。这些脚本可以在buildpack/bin文件夹中找到。
注意5：确保这3个脚本在buildpack/bin文件夹中具有执行权限。这些脚本以root用户身份在对等容器中执行，因此将权限更改为744就足够了。否则，将导致externalbuilder执行失败。
可在的org1文件夹中找到构建器的定义builders-config.yaml。core.yaml除了core_chaincode_externalbuilders选项（具有如下所示的自定义生成器配置）之外，此文件具有所有其他默认选项来配置对等项。

core.yaml中的ExternalBuilder定义
由于对等方的Kubernetes部署描述符具有用于配置选项的环境变量，因此这些环境变量将覆盖的默认选项core.yaml。之所以这样做是因为环境变量不接受数组格式，并且这种方式可以使用自己的配置来配置每个对等方，但是可以建立此构建器。该网络中的所有组织都使用它。
新的生命周期过程打包了链代码并以与以前版本不同的方式安装链代码。由于我们正在使用外部链码功能，因此链码代码不必在同一个Pod本身中进行编译和安装，而不必在另一个Pod上。在对等进程中唯一要安装的代码是能够连接到外部chaincode进程所需的信息。
注意6：我们将执行org1对等链代码的步骤，因此在org1 cli pod 上执行命令。对于org2对等方将重复这些步骤，但是稍后将更改链码的某些配置。
为此，我们需要将chaincode打包为一些要求。必须有一个connection.json文件，其中包含与外部链码服务器的连接信息。这包括地址，TLS证书和拨号超时配置。

对等链到外部链码的连接定义
该文件需要打包到名为的tar文件中code.tar.gz。通过执行以下命令来实现。
注意6：您可以在链式代码主机路径存储所指向的主机的cli工具窗格内或cli荚外部执行这些命令。最后，在cli pod中需要生成的tar文件来启动命令，因此我建议从cli pod中执行此操作。如果要在客户端中执行以下命令，请转到以下路径：/opt/gopath/src/github.com/marbles/packaging
$ cd链码/包装
$ tar cfz code.tar.gz connection.json
有了code.tar.gz文件后，我们必须再次将其与另一个名为的文件一起重新打包metadata.json，其中包括有关要处理的链码类型，链码驻留的路径以及我们要赋予该链码的标签的信息。

对等链到外部链码的元数据信息
我们使用以下命令打包这两个文件：
$ tar cfz marbles-org1.tgz code.tar.gz metadata.json
收到tar文件后，就可以使用新的生命周期过程将其安装在对等方中了。
$同行生命周期链代码安装marbles-org1.tgz
###应该打印类似的输出
2020-03-07 14：33：18.120 UTC [cli.lifecycle.chaincode] SubmitInstallProposal-> INFO 001远程安装：响应：<状态：200有效负载：“ \ nGdmarbles：e001937433673b11673d660d142c722fc372905db87f88d2448eee42c9c63064 006degree“> 
2020-03-07 14：33：18.126 UTC [cli.lifecycle.chaincode] SubmitInstallProposal-> INFO 002 Chaincode代码包标识符：marbles：e001937433673b11673d660d142c722fc372905db87f88d2448eee42c9c63064
复制链码代码包标识符，以备后用。不过，您始终可以通过执行以下命令来取回它：
$对等生命周期链代码查询已安装
###应该
在对等方上打印类似的输出已安装的链码：
包ID：大理石：030eec59c7d74fbb4e9fd57bbd50bb904a715ffb9de8fea85b6a6d4b8ca9ea12，标签：大理石
现在，我们将对org2重复上述相同的步骤，但是由于我们希望有一个不同的pod为org2对等方提供相同的链码，因此我们必须更改connection.json文件中的address config选项。像这样修改文件地址值：
“ address”：“ chaincode-marbles-org2.hyperledger：7052”，
然后重复与之前相同的步骤，在org2 cli pod中执行install命令：
$ rm -f code.tar.gz 
$ tar cfz code.tar.gz connection.json 
$ tar cfz marbles-org2.tgz code.tar.gz metadata.json 
$对等生命周期链码安装marbles-org2.tgz
###应该输出类似的输出
2020-03-07 15：10：15.093 UTC [cli.lifecycle.chaincode] SubmitInstallProposal-> INFO 001远程安装：响应：<状态：200有效负载：“ \ nGmarbles：c422c797444e4ee25a92a8eaf97765288a8d68f9c29cedf1e2cd \ e 006degree“> 
2020-03-07 15：10：15.093 UTC [cli.lifecycle.chaincode] SubmitInstallProposal-> INFO 002 Chaincode代码包标识符：marbles：c422c797444e4ee25a92a8eaf97765288a8d68f9c29cedf1e0cd82e4aa2c8a5b
像以前一样复制链码代码包标识符。当我们更改org2的地址时，它应具有与org1不同的哈希。
安装chaincode时对等端发生了什么？如果在对等端定义了一些外部构建器或启动器，则在尝试由对等端自身构建Docker容器的经典选项之前，它们将被执行。如我们所定义的external-builder，执行以下脚本顺序。
该detect脚本仅评估要安装的链码是否具有type键metadata.jsonas作为值external。如果该脚本失败，则对等方认为该外部构建器不必构建链码，并迭代到其他外部构建器，直到没有更多定义且回退到Docker构建过程为止。

检测externalbuilder的脚本
如果detect脚本成功，则build调用脚本。如果我们要在对等方内部构建链码，则此脚本应生成构件或二进制文件，但是由于我们希望将其作为外部服务使用，因此只需将connection.json文件复制到构建输出目录就足够了。

外部链码的构建脚本
最后，build脚本完成后，release将调用脚本。该脚本负责connection.json通过将其放置在发布输出目录中来提供对等端。因此，如果要执行该链码的某些调用，则对等方现在知道在何处调用该链码。

外部链码的发布脚本
构建和部署外部Chaincode
一旦在对等节点中安装了链码，就可以构建它们并将其部署到我们的Kubernetes集群中。让我们看一下我们需要对链码进行的更改。
由于链码不提供厂商，也不包含当前所需的模块，因此导入已更改，因此需要导入它们并提供模块。

导入外部链码
同样重要go.mod的是在构建代码之前定义文件以建立模块的版本。需要启用垫片模块中的最新版本才能启用“外部Chaincode”属性。

具有软件包所需版本的Go.mod
另一个更改是Chaincode Server属性，该属性侦听我们要建立的特定地址和端口。因为我们可能想通过使用Kubernetes yaml描述符来更改它们，所以我们将使用os 模块来获取地址的值和链码代码包标识符（CCID）。通过这样做，我们可以部署相同的代码映像，但是根据需要更改参数。

带有服务器实例定义的链码的主要功能
现在，我们准备使用以下命令构建chaincode容器Dockerfile：

链文件的Dockerfile
链码是使用golang高山图像构建的。构建阶段完成后，将二进制文件传输到裸露的高山映像中，以获取较小的安全映像。执行以下命令：
$ docker build -t chaincode / marbles：1.0。
如果一切正常，则应该已准备好部署映像。将链码部署的文件（org1-chaincode-deployment.yaml和org2-chaincode-deployment.yaml）CHAINCODE_CCID变量分别修改为在安装链码之前获得的值。

之后，部署它们：
$ kubectl创建-f chaincode / k8s
服务和部署将与其他工作负载部署在相同的k8s名称空间中。
$ kubectl get pods -n hyperledger
名称准备状态重新开始年龄
ca-org1-84945b8c7b-tx59g 1/1运行0 19h 
ca-org2-7454f69c48-nfzsq 1/1运行0 19h 
chaincode-marbles-org1-6fc8858855-wdz7z 1/1运行0 20m 
chaincode-marbles- org2-77bf56fdfb-6cdfm 1/1正在运行0 14m 
cli-org1-589944999c-cvgbx 1/1正在运行0 19h 
cli-org2-656cf8dd7c-kcxd7 1/1正在运行0 19h 
orderer0-5844bd9bcc-6td8c 1/1正在运行0 
46horderer1- 75d8df99cd-6vbjl 1/1运行中0 46h
orderer2-795cf7c4c-6lsdd 1/1运行0 46h 
peer0-org1-5bc579d766-kq2qd 1/1运行0 19h 
peer0-org2-77f58c87fd-sczp8 1/1运行0 19h
现在，我们必须批准每个组织的链码。这是链码生命周期过程的新功能，每个组织都必须同意批准链码的新定义。我们将批准大理石chaincode定义ORG1。在org1 cli pod中执行此命令，切记更改CHAINCODE_CCID：
$ peer lifecycle chaincode approveformyorg --channelID mychannel --name marbles --version 1.0 --init-required --package-id marbles：e001937433673b11673d660d142c722fc372905db87f88d2448eee42c9c63064 --sequence 1 -o orderer0：7050 --tls --cafile $ ORDERER_CA
###应该打印类似的输出
2020-03-08 10：02：46.192 UTC [chaincodeCmd] ClientWait-> INFO 001 txid [4d81ea5fd494e9717a0c860812d2b06bc62e4fc6c4b85fa6c3a916eee2c78e85]提交的状态为（VALID）
您可以通过执行以下命令来检查整个网络中的批准状态：
$对等生命周期链代码checkcommitreadiness --channelID mychannel --name marbles --version 1.0 --init-require 
d --sequence 1 -o -orderer0：7050 --tls --cafile $ ORDERER_CA
###应该
为组织'通道'mychannel'批准状态的链代码'marbles'版本'1.0'，序列'1' 输出类似的链代码定义，由org：
org1MSP：true 
org2MSP：false
现在，让我们批准org2。在org2 cli pod中执行此命令，切记更改CHAINCODE_CCID：
$ peer lifecycle chaincode approveformyorg --channelID mychannel --name marbles --version 1.0 --init-required --package-id marbles：25a9f6fe26161d29af928228ca1db0c41892e26e46335c84952336ee26d1fd93 --sequence 1 -o orderer0：7050 --tls --cafile $ ORDERER_
###应该打印类似的输出
2020-03-08 10：26：43.992 UTC [chaincodeCmd] ClientWait-> INFO 001 txid [74a89f3c93c10f14c626bd4d6cb654b37889908c9e6f7b983d2cad79f1e82267]已提交，状态为（VALID）
再次检查链码的提交准备情况：
$对等生命周期链代码checkcommitreadiness --channelID mychannel --name marbles --version 1.0 --init-required --sequence 1 -o orderer0：7050 --tls --cafile $ ORDERER_CA
###应该
为组织``mychannel''的通道代码'marbles'版本'1.0'，序列'1' 打印类似的输出链代码定义，org：
org1MSP：true 
org2MSP：true
现在，我们已获得所有组织的所有批准，让我们在渠道中提交此链码的定义。您可以在任何对等节点上执行此操作：
$ peer lifecycle chaincode commit -o orderer0：7050 --channelID mychannel --name marbles --version 1.0 --sequence 1 --init-required --tls true --cafile $ ORDERER_CA --peerAddresses peer0-org1：7051- tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2：7051 --tlsRootCertFiles / opt / gopath /src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt
###应该打印类似的输出
2020-03-08 14：13：49.516 UTC [chaincodeCmd] ClientWait-> INFO 001 txid [568cb81f821698025bbc61f4c6cd3b4baf1aea632e1e1a8cfdf3ec3902d1c6bd]在同等体0-03-08处以状态（VALID-08）提交，状态为（VALID-08）
14： 13：49.533 UTC [chaincodeCmd] ClientWait-> INFO 002 txid [568cb81f821698025bbc61f4c6cd3b4baf1aea632e1e1a1cf8fdf3ec3902d1c6bd]已提交，状态为（VALID）在peer0-org2：7051
现在，我们已将链码定义添加到通道，并准备被调用并对其进行查询！😄
测试外部链码
我们可以测试来自cli pod的调用和查询命令的链码。这些命令不会被生命周期链代码过程修改，可以在Hyperledger Fabric版本1.x中称为链代码。首先，让我们在分类帐中创建一些大理石。在一个客户端org1或org2上执行以下命令。
$同行链代码调用-o orderer0：7050 --isInit --tls true --cafile $ ORDERER_CA -C mychannel -n大理石--peerAddresses 
peer0-org1：7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger /fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses对等
0-org2：7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer /crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c'{“ Args”：[“ initMarble 
”，“ marble1”，“ blue”，“ 35”，“ tom”]}''- -waitForEvent
###应该打印类似的输出
2020-03-08 14：23：03.569 UTC [chaincodeCmd] ClientWait-> INFO 001 txid [83aeeaac47cf6302bc139addc4aa38116a40eaff788846d87cc815d2e1318f44]在peer0-org2：7051提交的状态为（VALID）
2020-03-08 14 23：03.575 UTC [chaincodeCmd] ClientWait-> INFO 002 txid [83aeeaac47cf6302bc139addc4aa38116a40eaff788846d87cc815d2e1318f44]在peer0-org1：7051 
2020-03-08 14：23：v.576上以状态（VALID）提交了状态（VALID）。结果：状态：200
这将在分类账上创建一个大理石1。现在创建另一个大理石：
$对等链代码调用-o orderer0：7050 --isInit --tls true --cafile $ ORDERER_CA -C mychannel -n marbles --peerAddresses peer0-org1：7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger /fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2：7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/ crypto / peerOrganizations / org2 / peers / peer0-org2 / tls / ca.crt -c'{“ Args”：[“ initMarble”，“ marble2”，“ red”，“ 50”，“ tom”]}'- waitForEvent
###应该打印类似的输出
2020-03-08 14：23：40.404 UTC [chaincodeCmd] ClientWait-> INFO 001 txid [8391f9f8ea84887a56f99e4dc4501eaa6696cd7bd6c524e4868bd6cfd5b85e78]在同位0-org2-03提交的状态为（VALID）
14： 23：40.434 UTC [chaincodeCmd] ClientWait-> INFO 002 txid [8391f9f8ea84887a56f99e4dc4501eaa6696cd7bd6c524e4868bd6cfd5b85e78]在对等0-组织1查询70链上以状态（VALID）提交状态（VALID）-代码
链>调用代码003-03md 14：23：40434。 。结果：状态：200
从marble1检索信息：
$对等链代码查询-C mychannel -n大理石-c'{“ Args”：[“ readMarble”，“ marble1”]}'
{“ docType”：“大理石”，“名称”：“大理石1”，“颜色”：“蓝色”，“尺寸”：35，“所有者”：“ tom”}
您可以使用此链代码执行许多示例命令，只需查看链代码的源代码以获取更多示例。
您还可以通过执行以下命令来检查chaincode容器的日志：
$ kubectl日志chaincode_pod_name -n hyperledger
###应该打印一个类似的输出
调用initMarble正在运行
-启动init marble- 
结束init marble 
调用正在运行initMarble- 
启动init marble- 
结束init marble 
运行在readMarble
奖金轨道！具有外部Chaincodes功能的基于ContractApi的Chaincodes
作为我们最近一直在玩的新功能，我们现在可以使用2.0版中引入的新ContractApi部署外部链码，这使得开发链码更加容易。我们将部署fabcar链代码，您可以在此处找到原始代码。修改后的文件将被推送到名为的文件夹下的仓库中fabcar。
我们将需要添加新的shim api，以允许添加允许与对等方进行外部通信的外部Chaincode服务器接口。

该go.mod文件也需要导入这两个模块。

这就是使它工作所需的所有修改！安装说明与之相同，只是使用fabcar（而不是大理石）一词。唯一的区别是，和中不需要--init-required或--isInit标志。approveformyorgcommitinvoke
对等生命周期链代码approveformyorg --channelID mychannel --name fabcar --version 1.0 --package-id fabcar：005c35f4f172c056723eca09d41e8048e0beaa2712d920c19af837640df7e2aa-序列1 -o orderer0：7050 --tls --cafile $ ORDERER_CA
对等生命周期链代码approveformyorg --channelID mychannel --name fabcar --version 1.0 --package-id fabcar：61ab817a6ad76098d340952e5d8e928d9c5ddff1a53341dc8d0c64b4345564b0-序列1 -o orderer0：7050 --tls --cafile $ ORDERER_CA
对等生命周期链代码checkcommitreadiness --channelID mychannel-名称fabcar-版本1.0-序列1 -o -orderer0：7050 --tls --cafile $ ORDERER_CA
对等生命周期链代码commit -o orderer0：7050 --channelID mychannel --name fabcar --version 1.0 --sequence 1 --tls true --cafile $ ORDERER_CA --peerAddresses peer0-org1：7051 --tlsRootCertFiles / opt / gopath / src / github.com / hyperledger / fabric / peer / crypto / peerOrganizations / org1 / peers / peer0-org1 / tls / ca.crt --peerAddresses peer0-org2：7051 --tlsRootCertFiles /opt/gopath/src/github.com /hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt
对等链代码调用-o orderer0：7050 --tls true --cafile $ ORDERER_CA -C mychannel -n fabcar --peerAddresses peer0-org1：7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer /crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2：7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/ org2 / peers / peer0-org2 / tls / ca.crt -c'{“ Args”：[“ InitLedger”]}'--waitForEvent
对等链代码查询-C mychannel -n fabcar -c'{“ Args”：[“ QueryAllCars”]}''
您还将在Hyperledger团队的github存储库中找到具有外部链码功能的相同链码的原始实现。它们的实现与我们的实现之间没有多少区别，因此我们假定这是使用外部功能实现基于契约的链码的正确方法。
未来的工作和改进
可以进行一些改进，我们很高兴收到社区的反馈，以改进这些外部链码。
外部Builders定义为环境变量而不是configmap。尽管它以这种方式工作，但我们希望将环境变量中的“外部构建器”定义为其他选项。我们试图将外部构建器定义为环境变量，但是由于预期ExternalBuilders配置为数组，并且环境变量仅接受字符串，因此存在很多错误。更新2：你们中的许多人都报告了许多问题，这些问题涉及外部构建器定义未成功执行或未正确检测到。这主要是由于对scripts文件夹的权限（从GitHub克隆后未启用执行权限）引起的。为了易于使用，buildpack脚本现已作为configmap包含在内。 并附加到具有更大kubernetes集成的对等Pod中。
背书策略失败：由于在通道上提交链码定义时，由于许多背书策略失败，我们不得不禁用了背书和生命周期背书策略。由于这可能需要更好地理解和定义2.0版中的策略，因此我们没有足够的时间正确配置它。更新1：现在，此问题已解决，并且对文章进行了新的更新，以使新的要求成为创始块中编码的认可策略和生命周期认可策略。
手动更改链码的地址配置。connection.json每当我们有一个新的组织时，都需要更改文件的地址值。如果我们每个组织有多个背书人，也可以更改。可能会有一些有趣的管道来帮助支持这些类型的链码的部署。
感谢您关注它！
