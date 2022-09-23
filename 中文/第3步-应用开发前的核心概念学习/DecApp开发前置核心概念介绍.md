在进行  dec app 应用开发之前，我们需要先了解一下与应用开发有关的几个重要概念。

## NamedObject(命名对象)

在  CYFS 网络中，NamedObject 是有具体意义，不可篡改，可被确权的对象数据。
网络中的许多操作都需要  NamedObject  才能进行，如发起请求时，请求参数必须是一个 NamedObject  对象，返回响应时，响应结果也必须是一个 NamedObject  对象。

## Zone

Zone 是由相同 Owner 的 Device 组成的星形结构，一般至少包含一个 OOD 和一定数量的 Device(也可以没有) ，OOD 是 Zone 内的数据集散中心，也是整个 Zone 对外通信的代表。
与传统的 C/S 架构类比，OOD 类似中心化服务器，Device 类似浏览器、手机等客户端。

## CYFS Stack

CYFS Stack 是 CYFS 协议的真正实现端。在使用 CYFS SDK 的 SharedCyfsStack 的各个接口时，实际上是向本地运行的 CYFS Stack 进程发起了一次基于 http 的 rpc 调用。

## MetaChain

CYFS 实现的区块链设施，它承担资产管理和可信数据的工作。
