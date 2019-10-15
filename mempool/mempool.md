# mempool

## 功能

**总的来说，mempool就是在内存中缓存一部分交易，这一部分交易在等待执行，共识，确认，最终加入到stoage中，下面是一些细节：**

* AC将验证过的交易发送到mempool，mempool中存储还未形成区块的交易
* mempool中每次加入新交易，都会转发给其他validator，让他们加入到自己的mempool中
* mempool每次收到别的validator发送的交易，会排序插入到自己的queue中。
* 为了减少带宽使用，我们只发送有可能被加入到下个区块的交易，（This means that either the sequence number of the transaction is the next sequence number of the sender account, or it is sequential to it.For example, if the current sequence number for an account is 2 and local mempool contains transactions with sequence numbers 2, 3, 4, 7, 8, then only transactions 2, 3, and 4 will be broadcast）.同时，我们不会转发其他validator转发来的交易，这些validator自己负责转发给所有的validator
* mempool中的交易组成区块通过consenseus与其他validator达成一致，mempool不会主动发送交易给consensus，而是consensus主动pull mempool中的交易
* mempool中的交易根据gas price排序（gloabal，在保证正确的前提下激励人们用gas price换取更快进入block。在同一账户发起的交易中，按sequence number排序，
* 当一笔交易被写进storage中后，consensus模块会给mempool发送信息来噶奥苏他不用继续存储这笔交易了
* 为防止攻击，mempool中的交易数目和每个交易的存在时间都有限制，时间限制为：systemTTL and client-specified expiration，超过之后会被垃圾回收。

## 与其他模块的交互

与VM，consensus，其他validator存在交互

## 其他细节

见[Libra官方关于mempool的文档](https://developers.libra.org/docs/crates/mempool)

## 目录

    mempool/src
    ├── core_mempool             # main in memory data structure
    ├── proto                    # protobuf definitions for interactions with mempool
    ├── lib.rs
    ├── mempool_service.rs       # gRPC service
    ├── runtime.rs               # bundle of shared mempool and gRPC service
    └── shared_mempool.rs        # shared mempool
## 具体实现

详见各个md模块中



