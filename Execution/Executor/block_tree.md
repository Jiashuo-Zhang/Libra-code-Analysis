# block_tree

##含义：

```rust
//! In a leader based consensus algorithm, each participant maintains a block tree that looks like
//! the following:
//! ```text
//!  Height      5      6      7      ...
//!
//! Committed -> B5  -> B6  -> B7
//!         |
//!         └--> B5' -> B6' -> B7'
//!                     |
//!                     └----> B7"
//! ```
//! This module implements `BlockTree` that is an in-memory representation of this tree.
```

## 功能：

这一部分实现了树的各种操作，比如看看某个block的儿子，父亲，插入block等等操作，

## 代码：

####定义：

  ```rust
//定义block的trait
//值得注意的set_committed之后可能不会马上加入storage，因为execution操作可能会滞后一点点。

pub trait Block: std::fmt::Debug {
    /// The output of executing this block.
    type Output;

    /// The signatures on this block.
    type Signature;

    /// Whether consensus has decided to commit this block. This kind of blocks are expected to be
    /// sent to storage very soon, unless execution is lagging behind.
    fn is_committed(&self) -> bool;

    /// Marks this block as committed.
    fn set_committed(&mut self);

    /// Whether this block has finished execution.
    fn is_executed(&self) -> bool;

    /// Sets the output of this block.
    fn set_output(&mut self, output: Self::Output);

    /// Sets the signatures for this block.
    fn set_signature(&mut self, signature: Self::Signature);

    /// The id of this block.
    fn id(&self) -> HashValue;

    /// The id of the parent block.
    fn parent_id(&self) -> HashValue;

    /// Adds a block as its child.
    fn add_child(&mut self, child_id: HashValue);

    /// The list of children of this block.
    fn children(&self) -> &HashSet<HashValue>;
}



/// The `BlockTree` implementation.
#[derive(Debug)]
pub struct BlockTree<B> {
    /// A map that keeps track of all existing blocks by their ids.
    id_to_block: HashMap<HashValue, B>,

    /// The blocks at the lowest height in the map. B5 and B5' in the following example.
    /// ```text
    /// Committed(B0..4) -> B5  -> B6  -> B7
    ///                |
    ///                └--> B5' -> B6' -> B7'
    ///                            |
    ///                            └----> B7"
    /// ```
    heads: HashSet<HashValue>,

    /// Id of the last committed block. B4 in the above example.
    last_committed_id: HashValue,
}/// The `BlockTree` implementation.
#[derive(Debug)]
pub struct BlockTree<B> {
    /// A map that keeps track of all existing blocks by their ids.
    id_to_block: HashMap<HashValue, B>,
/////每一个block的id都是一个hash
    
    /// The blocks at the lowest height in the map. B5 and B5' in the following example.
    /// ```text
    /// Committed(B0..4) -> B5  -> B6  -> B7
    ///                |
    ///                └--> B5' -> B6' -> B7'
    ///                            |
    ///                            └----> B7"
    /// ```
    heads: HashSet<HashValue>,

    /// Id of the last committed block. B4 in the above example.
    last_committed_id: HashValue,
}
  ```

### 几个比较有意思的函数；除了这几个，剩下的都比较trivial

```rust
/// Returns a reference to a specific block, if it exists in the tree.
    pub fn get_block(&self, id: HashValue) -> Option<&B> {
        self.id_to_block.get(&id)
    }

    /// Returns a mutable reference to a specific block, if it exists in the tree.
///这个函数往往是其他需要修改树的函数的基础，因为要获取可变的引用
    pub fn get_block_mut(&mut self, id: HashValue) -> Option<&mut B> {
        self.id_to_block.get_mut(&id)
    }



//必须要父节点execute完了才能excute自己
/// Returns id of a block that is ready to be sent to VM for execution (its parent has finished
    /// execution), if such block exists in the tree.
    pub fn get_block_to_execute(&mut self) -> Option<HashValue> {
        let mut to_visit: Vec<HashValue> = self.heads.iter().cloned().collect();

        while let Some(id) = to_visit.pop() {
            let block = self
                .id_to_block
                .get(&id)
                .expect("Missing block in id_to_block.");
            if !block.is_executed() {
                return Some(id);
            }
            to_visit.extend(block.children().iter().cloned());
        }

        None
    }



//如果自己被确认了，那么自己的所有祖先节点也都被确认了，这些区块都将会被送到storage永久的存储，但可能不是马上完成操作
 /// Marks given block and all its uncommitted ancestors as committed. This does not cause these
    /// blocks to be sent to storage immediately.
    pub fn mark_as_committed(
        &mut self,
        id: HashValue,
        signature: B::Signature,
    ) -> Result<(), CommitBlockError> {
        // First put the signatures in the block. Note that if this causes multiple blocks to be
        // marked as committed, only the last one will have the signatures.
        
        //也就是说，有的被确认的区块会不存在数字签名
        match self.id_to_block.get_mut(&id) {
            Some(block) => {
                if block.is_committed() {
                    bail_err!(CommitBlockError::BlockAlreadyMarkedAsCommitted { id });
                } else {
                    block.set_signature(signature);
                }
            }
            None => bail_err!(CommitBlockError::BlockNotFound { id }),
        }

        // Mark the current block as committed. Go to parent block and repeat until a committed
        // block is found, or no more blocks.
        let mut current_id = id;
        while let Some(block) = self.id_to_block.get_mut(&current_id) {
            if block.is_committed() {
                break;
            }

            block.set_committed();
            current_id = block.parent_id();
        }
/////向上递推地commit
        Ok(())
    } 



 /// Removes all blocks in the tree that conflict with committed blocks. Returns a list of
    /// blocks that are ready to be sent to storage (all the committed blocks that have been
    /// executed).
    pub fn prune(&mut self) -> Vec<B> {
        let mut blocks_to_store = vec![];

        // First find if there is a committed block in current heads. Since these blocks are at the
        // same height, at most one of them can be committed. If all of them are pending we have
        // nothing to do here.  Otherwise, one of the branches is committed. Throw away the rest of
        // them and advance to the next height.
        let mut current_heads = self.heads.clone();
        while let Some(committed_head) = self.get_committed_head(&current_heads) {
            assert!(
                current_heads.remove(&committed_head),
                "committed_head should exist.",
            );
            for id in current_heads {
                self.remove_branch(id);
            }

            match self.id_to_block.entry(committed_head) {
                hash_map::Entry::Occupied(entry) => {
                    current_heads = entry.get().children().clone();
                    let current_id = *entry.key();
                    let parent_id = entry.get().parent_id();
                    if entry.get().is_executed() {
                        // If this block has been executed, all its proper ancestors must have
                        // finished execution and present in `blocks_to_store`.
                        self.heads = current_heads.clone();
                        self.last_committed_id = current_id;
                        blocks_to_store.push(entry.remove());
                    } else {
                        // The current block has not finished execution. If the parent block does
                        // not exist in the map, that means parent block (also committed) has been
                        // executed and removed. Otherwise self.heads does not need to be changed.
                        if !self.id_to_block.contains_key(&parent_id) {
                            self.heads = HashSet::new();
                            self.heads.insert(current_id);
                        }
                    }
                }
                hash_map::Entry::Vacant(_) => unreachable!("committed_head_id should exist."),
            }
        }

        blocks_to_store
    }

```

