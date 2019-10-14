# VM_validator.rs

## 功能：

**提供一个对交易进行验证的服务，判断其是否合法值得注意的是这一部分并没有实现validate_transactions的逻辑，只是简单的进行了包装，调用MoveVM的一个函数（vm.validate_transaction)实现的，具体实现中，会有输入一个相应的state_view（包含账户信息等）和一个txn，通过这两个参数进行判断。**

## 模块：

* 主要功能在vm_validator.rs 中实现

## 与其他模块的交互

* AC会生成VM_validator的instance
* 会与storage和VM模块进行交互

## 代码：

#### 定义

```rust
 //定义了一个trait，一个结构体
pub trait TransactionValidation: Send + Sync {
    type ValidationInstance: VMVerifier;
    /// Validate a txn from client
    fn validate_transaction(
        &self,
        _txn: SignedTransaction,
    ) -> Box<dyn Future<Item = Option<VMStatus>, Error = failure::Error> + Send>;
}

#[derive(Clone)]
//结构体中有一个读storage的client，一个用于验证的虚拟机
pub struct VMValidator {
    storage_read_client: Arc<dyn StorageRead>,////包含一个storage client，来获得相应账户的sequence，balance等信息
    vm: MoveVM,
}

impl VMValidator {
    pub fn new(config: &NodeConfig, storage_read_client: Arc<dyn StorageRead>) -> Self {
        VMValidator {
            storage_read_client,
            vm: MoveVM::new(&config.vm_config),
        }
    }
}

```

#### VMvalidator实现transactionvalidation接口

```rust
impl TransactionValidation for VMValidator {
    type ValidationInstance = MoveVM;

    fn validate_transaction(
        &self,
        txn: SignedTransaction,
    ) -> Box<dyn Future<Item = Option<VMStatus>, Error = failure::Error> + Send> {
        // TODO: For transaction validation, there are two options to go:
        // 1. Trust storage: there is no need to get root hash from storage here. We will
        // create another struct similar to `VerifiedStateView` that implements `StateView`
        // but does not do verification.
        // 2. Don't trust storage. This requires more work:
        // 1) AC must have validator set information
        // 2) Get state_root from transaction info which can be verified with signatures of
        // validator set.
        // 3) Create VerifiedStateView with verified state
        // root.

        // Just ask something from storage. It doesn't matter what it is -- we just need the
        // transaction info object in account state proof which contains the state root hash.
        let address = AccountAddress::new([0xff; ADDRESS_LENGTH]);
        let item = RequestItem::GetAccountState { address };

        match self
            .storage_read_client
            .update_to_latest_ledger(/* client_known_version = */ 0, vec![item])
        {	//这里应该是用了上面函数的默认参数值，功能就是查找vec包含的内容，然后返回状态（详见AC部分）
            //获取account_state，并进行一系列检查
            Ok((mut items, _, _)) => {
                if items.len() != 1 {
                    return Box::new(err(format_err!(
                        "Unexpected number of items ({}).",
                        items.len()
                    )
                    .into()));
                }

                match items.remove(0) {
                    ResponseItem::GetAccountState {
                        account_state_with_proof,
                    } => {
                        let transaction_info = account_state_with_proof.proof.transaction_info();
                        let state_root = transaction_info.state_root_hash();
                        let smt = SparseMerkleTree::new(state_root);
                        let state_view = VerifiedStateView::new(
                            Arc::clone(&self.storage_read_client),
                            state_root,
                            &smt,
                        );
                        //包装相关账户的状态
                        Box::new(ok(self.vm.validate_transaction(txn, &state_view)))
                        //把相关账户的状态和带验证的交易传入虚拟机，虚拟机运行不报错则表示交易合法
                    }
                    _ => panic!("Unexpected item in response."),
                }
            }
            Err(e) => Box::new(err(e.into())),//Box是一个智能指针
        }
    }
}
```

#### get_account_state :获取账户的sequence number和balance

```rust
pub async fn get_account_state(
    storage_read_client: Arc<dyn StorageRead>,
    address: AccountAddress,
) -> Result<(u64, u64)> {
    let req_item = RequestItem::GetAccountState { address };
    let (response_items, _, _) = storage_read_client
        .update_to_latest_ledger_async(0 /* client_known_version */, vec![req_item])
        .await?;
    // 类似上面函数的注释，返回一个函数的状态和proof
    let account_state = match &response_items[0] {
        ResponseItem::GetAccountState {
            account_state_with_proof,
        } => &account_state_with_proof.blob,
        _ => bail!("Not account state response."),
    };
    let account_resource = get_account_resource_or_default(account_state)?;
    let sequence_number = account_resource.sequence_number();
    let balance = account_resource.balance();
    Ok((sequence_number, balance))
}
```

## 总结： 

**这一部分体现了封装的基本思想。这一部分起到的作用就是通过底层的查询account状态的函数，组装出相关参数，然后调用VM中实现的函数，最终完成调用功能**



