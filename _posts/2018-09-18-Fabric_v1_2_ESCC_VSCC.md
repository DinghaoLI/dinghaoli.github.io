---
layout: post
title:  "Hyperledger Fabric v1.2 新特性 - 可插拔ESCC/VSCC"
date:   2018-09-18 00:00:00 +0800
categories: Blockchain
tags: Blockchain

---

### 一、背景与需求

 一般来说，System chaincode需要在peer启动的时候被部署和注册，而且无法做到热替换 (如果更新或者替换，需要重启peer)。不同的chaincode可能需要不同的ESCC和VSCC来保证交易的合法性。有时，根据业务场景的不同，我们可能需要去实现自定义的ESCC/VSCC，而不仅仅是使用Fabic默认的策略，比如：

- 1、基于状态的背书：背书策略有时候会和state中的key相关联。
- 2、UTXO的模型：验证账户是否有Double-spending的现象。
- 3、匿名交易场景：当背书不包括peer的identity，而只有一个签名和一个认证的匿名公钥时。

### 二、实现与部署


所有的可插拔E/VSCC的模块以golang plugin（.so文件）的形式实现并且配置在core.yaml文件中，且.so文件需要在peer的本地存放。   

e.g.

```

handlers:
    endorsers:
      escc:
        name: DefaultEndorsement
      statebased:
        name: state_based
        library: /etc/hyperledger/fabric/plugins/state_based_endorsement.so
    validators:
      vscc:
        name: DefaultValidation
      statebased:
        name: state_based
        library: /etc/hyperledger/fabric/plugins/state_based_validation.so

```

### 三、ESCC插件的实现


ESCC必须实现 core/handlers/endorsement/api/endorsement.go 文件中的Plugin interface。

```
// Plugin endorses a proposal response
type Plugin interface {
    // Endorse signs the given payload(ProposalResponsePayload bytes), and optionally mutates it.
    // Returns:
    // The Endorsement: A signature over the payload, and an identity that is used to verify the signature
    // The payload that was given as input (could be modified within this function)
    // Or error on failure
    Endorse(payload []byte, sp *peer.SignedProposal) (*peer.Endorsement, []byte, error)

    // Init injects dependencies into the instance of the Plugin
    Init(dependencies ...Dependency) error
}
```


同时，PluginFactory接口中的New方法为每个channel创建插件，该方法也应该由插件开发人员实现：


```
// PluginFactory creates a new instance of a Plugin
type PluginFactory interface {
    New() Plugin
}
```


Init(dependencies …Dependency) 方法可以接受满足Denpendency接口的输入，当前的Fabric支持如下几种dependency作为ESCC plugin的输入：


- 1、SigningIdentityFetcher: 根据被签名的proposal返回一个SigningIdentity实例

```
// SigningIdentity signs messages and serializes its public identity to bytes
type SigningIdentity interface {
    // Serialize returns a byte representation of this identity which is used to verify
    // messages signed by this SigningIdentity
    Serialize() ([]byte, error)

    // Sign signs the given payload and returns a signature
    Sign([]byte) ([]byte, error)
}
```


- 2、StateFetche: 获取一个State对象，用于和world state的交互

```
// State defines interaction with the world state
type State interface {
    // GetPrivateDataMultipleKeys gets the values for the multiple private data items in a single call
    GetPrivateDataMultipleKeys(namespace, collection string, keys []string) ([][]byte, error)

    // GetStateMultipleKeys gets the values for multiple keys in a single call
    GetStateMultipleKeys(namespace string, keys []string) ([][]byte, error)

    // GetTransientByTXID gets the values private data associated with the given txID
    GetTransientByTXID(txID string) ([]*rwset.TxPvtReadWriteSet, error)

    // Done releases resources occupied by the State
    Done()
 }
```

### 四、 VSCC插件的实现

VSCC必须实现 core/handlers/validation/api/validation.go 文件中的Plugin interface

```
// Plugin validates transactions
type Plugin interface {
    // Validate returns nil if the action at the given position inside the transaction
    // at the given position in the given block is valid, or an error if not.
    Validate(block *common.Block, namespace string, txPosition int, actionPosition int, contextData ...ContextDatum) error

    // Init injects dependencies into the instance of the Plugin
    Init(dependencies ...Dependency) error
}
```

对于VSCC，开发人员也同样需要实现PluginFactory

```
// PluginFactory creates a new instance of a Plugin
type PluginFactory interface {
    New() Plugin
}
```

对于VSCC的dependency，目前Fabric支持的有：

- 1、IdentityDeserializer：将byte representation of identities转换为可用验证签名的Identity对象，并根据其对应的MSP进行验证。

- 2、PolicyEvaluator：判断policy是否被满足

```
// PolicyEvaluator evaluates policies
type PolicyEvaluator interface {
    validation.Dependency

    // Evaluate takes a set of SignedData and evaluates whether this set of signatures satisfies
    // the policy with the given bytes
    Evaluate(policyBytes []byte, signatureSet []*common.SignedData) error
}
```


- 3、StateFetcher：获取一个State对象，用于和world state的交互

```
// State defines interaction with the world state
type State interface {
    // GetStateMultipleKeys gets the values for multiple keys in a single call
    GetStateMultipleKeys(namespace string, keys []string) ([][]byte, error)

    // GetStateRangeScanIterator returns an iterator that contains all the key-values between given key ranges.
    // startKey is included in the results and endKey is excluded. An empty startKey refers to the first available key
    // and an empty endKey refers to the last available key. For scanning all the keys, both the startKey and the endKey
    // can be supplied as empty strings. However, a full scan should be used judiciously for performance reasons.
    // The returned ResultsIterator contains results of type *KV which is defined in protos/ledger/queryresult.
    GetStateRangeScanIterator(namespace string, startKey string, endKey string) (ResultsIterator, error)

    // Done releases resources occupied by the State
    Done()
}
```


### 五、VSCC的一致性

目前VSCC plugin在各个节点的一致性需要开发人员自己来保证。在Fabric未来的版本中，会能达到VSCC一致性的检验，让在每一个peer中的VSCC都是相同的版本。VSCC plugin再无法验证交易的时候（比如再无法访问数据库的时候），需要抛出错误，类型为ExecutionFailureError。为了保证一致性，此交易将会被认为是不合法的。












