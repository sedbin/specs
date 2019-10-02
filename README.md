# Sedbin

Sedbin is a concept of proof of work blockchain with shared
security. Similar to merge mining, it allows multiple blockchains to
use the same source of PoW, thus increases the security and better
protects against 51% attack. Sedbin, however, has a smaller seal size,
and requires much less logic changes to participating blockchains. It
also enables security sharing for three or more blockchains, instead
of just two in the situation of merge mining.

Sedbin is also permissionless, in terms of participating
blockchains. They can join or leave any sedbin at any times. Low miner
participation of a *participating blockchain* will results in larger
seal proof size, and longer block time. However, the security of any
participating blockchain will not be impacted. 

Participating blockchains function by itself, and do not require
additional data from master blockchain for validation.

## Design

Sedbin consists of a master blockchain, and a number of participating
blockchains. Those participating blockchains uses master blockchain's
proof of work, and shares security.

Master blockchain's header contains a trie root, in which a trie
structure allows list of arbitrary data to be recorded. Those data are
expected to be the pre-hash of participating blockchain's header. Once
a pre-hash is recorded in master blockchain's header, participating
blockchains can use that header to generate a valid seal, and produce
a new block.

It is expected that there are multiple sedbins. Each sedbin has a
particular PoW mining algorithm, a difficulty adjustment algorithm,
and a fork choice rule. All participating blockchains of one sedbin
shares the same mining algorithm and block time.

No coins are involved in master blockchain. The incentive for mining
master blockchain is solely for the rewards of all participating
blockchains.

## Specification

For a sedin, define `Hash`, which is the hash output type, given the
hash function used on the master blockchain. We then define the header
structure for master blockchain, encoded in SCALE.

```rust
struct Header {
  parent_hash: Hash,
  data_size: u32,
  data_root: Hash,
  seal: Seal,
}
```

`Seal` is a data structure specific for a sedbin, used to encode the
proof of work. For example, a common seal structure would be `(nonce,
difficulty, work)`.

`data_size` and `data_root` collectively forms a binary merkle tree
where it has a list of data (of length `data_size`) where all of the
data are of type `Hash`.

The master blockchain also defines its own difficulty adjustment rules
and fork choice rules.

For a participating blockchain, it can generate its seal as followed,
once its pre-hash is recorded in the master blockchain.

```rust
struct ParticipatingSeal {
  skipped: Vec<Header>,
  master: Header,
  index: u32,
  proofs: Vec<Hash>,
}
```

`skipped` is a list of headers that has been skipped since last block
on participating blockchain. `master` is the current block of master
blockchain where the `pre-hash` is recorded. `index` and `proofs` form
a normal binary merkle tree proof.

If participating blockchain's pre-hash has a different size compared
with `Hash` on master blockchain, then first hash the pre-hash (using
master blockchain's hash algorithm).

Participating blockchain's fork choice is as follows -- choose the
current canonical chain on master blockchain, find participating
blockchain's block pre-hashes on the chain, skipping any invalid or
unknown data on the master blockchain.

## Variants

Below we define some concrete sedbins with given PoW algorithm,
difficulty adjustment rules and fork choice rules.

### Ethash Sedbin

With `Hash` being `H256`, mining algorithm being Ethash. Seal being
Ethash seal with difficulty `(nonce: H64, difficulty: U256, mix_hash:
H256)`. The difficulty adjustment algorithm being the one with EIP-100
applied. Use largest total difficulty as the fork choice rule.

### RandomX Sedbin

With `Hash` being `H256`, mining algorithm being RandomX. Seal being
the data structure `(difficulty: U256, work: H256, nonce: H256)`.

The difficulty adjustment algorithm takes `DIFFICULTY_ADJUST_WINDOW`
block's block time:

```rust
pub fn damp(actual: u128, goal: u128, damp_factor: u128) -> u128 {
	(actual + (damp_factor - 1) * goal) / damp_factor
}

pub fn clamp(actual: u128, goal: u128, clamp_factor: u128) -> u128 {
	max(goal / clamp_factor, min(actual, goal * clamp_factor))
}

fn difficulty(
  past_data: &[DifficultyAndTimestamp; DIFFICULTY_ADJUST_WINDOW]
) -> U256 {
  let mut ts_delta = 0;
  for i in 1..(DIFFICULTY_ADJUST_WINDOW as usize) {
    let prev = data[i - 1].map(|d| d.timestamp);
    let cur = data[i].map(|d| d.timestamp);

    let delta = match (prev, cur) {
      (Some(prev), Some(cur)) => cur.saturating_sub(prev),
      _ => BLOCK_TIME_MSEC.into(),
    };
    ts_delta += delta;
  }

  if ts_delta == 0 {
    ts_delta = 1;
  }

  let mut diff_sum = U256::zero();
  for i in 0..(DIFFICULTY_ADJUST_WINDOW as usize) {
    let diff = match data[i].map(|d| d.difficulty) {
      Some(diff) => diff,
      None => InitialDifficulty::get(),
    };
    diff_sum += diff;
  }

  if diff_sum < U256::from(MIN_DIFFICULTY) {
    diff_sum = U256::from(MIN_DIFFICULTY);
  }

  let adj_ts = clamp(
    damp(ts_delta, BLOCK_TIME_WINDOW_MSEC as u128, DIFFICULTY_DAMP_FACTOR),
    BLOCK_TIME_WINDOW_MSEC as u128,
    CLAMP_FACTOR,
  );

  min(U256::from(MAX_DIFFICULTY),
      max(U256::from(MIN_DIFFICULTY),
      diff_sum * U256::from(BLOCK_TIME_MSEC) / U256::from(adj_ts)))
}
```
