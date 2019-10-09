# Accumulator

## 概念：

**Merkle Accumulator就是一棵Merkle树，想象一个merkle树的结构，每一个父节点都是两个子节点的hash，这就要求子节点必须成对出现，不能出现一个节点的出度是1的情况，但是在插入的过程中一定会出现这种情况，为此，我们使用Placeholder Nodes来填补空白**

**Merkle Accumulator还把节点分成了 Frozen Nodes & Non-frozen Nodes，Frozen的意思就是这个节点的值是不会变的，也就是说他的子树的所有结构都已经固定了，不可能随着插入节点而变化了，也就是说他的子树里面不能含有Placeholder Nodes **

**这一部分是storage实现的最底层的部分，其他的比如eventstroage等都是基于这个模块实现的 **

## 代码：

####定义了几个接口和结构体，比较好理解。

```rust
/// Defines the interface between `MerkleAccumulator` and underlying storage.
pub trait HashReader {
    /// Return `HashValue` carried by the node at `Position`.
    fn get(&self, position: Position) -> Result<HashValue>;
}

/// A `Node` in a `MerkleAccumulator` tree is a `HashValue` at a `Position`
type Node = (Position, HashValue);

/// In this live Merkle Accumulator algorithms.
pub struct MerkleAccumulator<R, H> {
    reader: PhantomData<R>,
    hasher: PhantomData<H>,
}

impl<R, H> MerkleAccumulator<R, H>
where
    R: HashReader,
    H: CryptoHasher,
{
    /// Given an existing Merkle Accumulator (represented by `num_existing_leaves` and a `reader`
    /// that is able to fetch all existing frozen nodes), and a list of leaves to be appended,
    /// returns the result root hash and new nodes to be frozen.
    pub fn append(
        reader: &R,
        num_existing_leaves: u64,
        new_leaves: &[HashValue],
    ) -> Result<(HashValue, Vec<Node>)> {
        MerkleAccumulatorView::<R, H>::new(reader, num_existing_leaves).append(new_leaves)
    }

    /// Get proof of inclusion of the leaf at `leaf_index` in this Merkle Accumulator of
    /// `num_leaves` leaves in total. Siblings are read via `reader` (or generated dynamically
    /// if they are non-frozen).
    ///
    /// See [`types::proof::AccumulatorProof`] for proof format.
    pub fn get_proof(reader: &R, num_leaves: u64, leaf_index: u64) -> Result<AccumulatorProof> {
        MerkleAccumulatorView::<R, H>::new(reader, num_leaves).get_proof(leaf_index)
    }
}
```

#### 实际实现：

 ```rust
/// Actual implementation of Merkle Accumulator algorithms, which carries the `reader` and
/// `num_leaves` on an instance for convenience
struct MerkleAccumulatorView<'a, R, H> {
    reader: &'a R,
    num_leaves: u64,
    hasher: PhantomData<H>,
}

impl<'a, R, H> MerkleAccumulatorView<'a, R, H>
where
    R: HashReader,
    H: CryptoHasher,
{
    fn new(reader: &'a R, num_leaves: u64) -> Self {
        Self {
            reader,
            num_leaves,
            hasher: PhantomData,
        }
    }

    /// implementation for pub interface `MerkleAccumulator::append`
    ////实现了append的操作，如果不看也没关系，理解干了什么就行
    fn append(&self, new_leaves: &[HashValue]) -> Result<(HashValue, Vec<Node>)> {
        // Deal with the case where new_leaves is empty
        if new_leaves.is_empty() {
            if self.num_leaves == 0 {
                return Ok((*ACCUMULATOR_PLACEHOLDER_HASH, Vec::new()));
            } else {
                let root_hash = self.get_hash(Position::get_root_position(self.num_leaves - 1))?;
                return Ok((root_hash, Vec::new()));
            }
        }

        let num_new_leaves = new_leaves.len();
        let last_new_leaf_idx = self.num_leaves + num_new_leaves as u64 - 1;
        let root_level = Position::get_root_position(last_new_leaf_idx).get_level() as usize;
        let mut to_freeze = Vec::with_capacity(Self::max_to_freeze(num_new_leaves, root_level));

        // create one new node for each new leaf hash
        let mut current_level = self.gen_leaf_level(new_leaves);
        Self::record_to_freeze(
            &mut to_freeze,
            &current_level,
            false, /* has_non_frozen */
        );

        // loop starting from leaf level, upwards till root_level - 1,
        // making new nodes of parent level and recording frozen ones.
        //从叶节点递推到根节点，在这过程中，检查是不是使用了placeholder，一旦使用了，那么在这之上的所有节点都会变成non forzen的，值得注意的死活，每一次的gen-parent-level是一层一层进行的，即每次生成一整个level
        let mut has_non_frozen = false;
        for _ in 0..root_level {
            let (parent_level, placeholder_used) = self.gen_parent_level(&current_level)?;

            // If a placeholder node is used to generate the right most node of a certain level,
            // such level and all its parent levels have a non-frozen right most node.
            has_non_frozen |= placeholder_used;
            Self::record_to_freeze(&mut to_freeze, &parent_level, has_non_frozen);

            current_level = parent_level;
        }

        assert_eq!(current_level.len(), 1, "must conclude in single root node");
        Ok((current_level.first().expect("unexpected None").1, to_freeze))
    }
    
    
    
    
    
    
    /// upper bound of num of frozen nodes:
    ///     new leaves and resulting frozen internal nodes forming a complete binary subtree
    ///         num_new_leaves * 2 - 1 < num_new_leaves * 2
    ///     and the full route from root of that subtree to the accumulator root turns frozen
    ///         height - (log2(num_new_leaves) + 1) < height - 1 = root_level
    fn max_to_freeze(num_new_leaves: usize, root_level: usize) -> usize {
        num_new_leaves * 2 + root_level
    }
    

    fn hash_internal_node(left: HashValue, right: HashValue) -> HashValue {
        MerkleTreeInternalNode::<H>::new(left, right).hash()
    }

    
    /// Given leaf level hashes, create leaf level nodes
    fn gen_leaf_level(&self, new_leaves: &[HashValue]) -> Vec<Node> {
        new_leaves
            .iter()
            .enumerate()
            .map(|(i, hash)| (Position::from_leaf_index(self.num_leaves + i as u64), *hash))
            .collect()
    }

    /// Given a level of new nodes (frozen or not), return new nodes on its parent level, and
    /// a boolean value indicating whether a placeholder node is used to construct the last node
    //只有last node才会可能使用place holder（凑整）
    fn gen_parent_level(&self, current_level: &[Node]) -> Result<((Vec<Node>, bool))> {
        let mut parent_level: Vec<Node> = Vec::with_capacity(current_level.len() / 2 + 1);
        let mut iter = current_level.iter().peekable();

        // first node may be a right child, in that case pair it with its existing sibling
        let (first_pos, first_hash) = iter.peek().expect("Current level is empty");
        if first_pos.get_direction_for_self() == NodeDirection::Right {
            parent_level.push((
                first_pos.get_parent(),
                Self::hash_internal_node(self.reader.get(first_pos.get_sibling())?, *first_hash),
            ));
            iter.next();
        }

        // walk through in pairs of siblings, use placeholder as last right sibling if necessary
        let mut placeholder_used = false;
        while let Some((left_pos, left_hash)) = iter.next() {
            let right_hash = match iter.next() {
                Some((_, h)) => h,
                None => {
                    placeholder_used = true;
                    &ACCUMULATOR_PLACEHOLDER_HASH
                }
            };

            parent_level.push((
                left_pos.get_parent(),
                Self::hash_internal_node(*left_hash, *right_hash),
            ));
        }

        Ok((parent_level, placeholder_used))
    }

    /// append a level of new nodes into output vector, skip the last one if it's a non-frozen node
    fn record_to_freeze(to_freeze: &mut Vec<Node>, level: &[Node], has_non_frozen: bool) {
        to_freeze.extend(
            level
                .iter()
                .take(level.len() - has_non_frozen as usize)
                .cloned(),
        )
    }

    fn get_hash(&self, position: Position) -> Result<HashValue> {
        if position.is_placeholder(self.num_leaves - 1) {
            Ok(*ACCUMULATOR_PLACEHOLDER_HASH)
        } else if position.is_freezable(self.num_leaves - 1) {
            self.reader.get(position)
        } else {
            // non-frozen non-placeholder node
            Ok(Self::hash_internal_node(
                self.get_hash(position.get_left_child())?,
                self.get_hash(position.get_right_child())?,
            ))
        }
    }

    /// implementation for pub interface `MerkleAccumulator::get_proof`
    fn get_proof(&self, leaf_index: u64) -> Result<AccumulatorProof> {
        ensure!(
            leaf_index < self.num_leaves,
            "invalid leaf_index {}, num_leaves {}",
            leaf_index,
            self.num_leaves
        );

        let leaf_pos = Position::from_leaf_index(leaf_index);
        let root_pos = Position::get_root_position(self.num_leaves - 1);

        let siblings: Vec<HashValue> = leaf_pos
            .iter_ancestor_sibling()
            .take(root_pos.get_level() as usize)
            .map(|p| self.get_hash(p))
            .collect::<Result<Vec<HashValue>>>()?
            .into_iter()
            .rev()
            .collect();

        Ok(AccumulatorProof::new(siblings))
    }
}
 ```





