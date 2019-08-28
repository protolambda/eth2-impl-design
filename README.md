# Ethereum 2.0 Implementation Design

[![Join the chat at https://discord.gg/hpFs23p](https://img.shields.io/badge/chat-on%20discord-blue.svg)](https://discord.gg/hpFs23p) [![Join the chat at https://gitter.im/ethereum/sharding](https://badges.gitter.im/ethereum/sharding.svg)](https://gitter.im/ethereum/sharding?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

This repository documents client implementation design choices, without making them part of the official documentation.

Maintained by @protolambda. Suggestions welcome.

----

## Phase 0

### State transition optimizations

Spec: [`ethereum/eth2.0-specs/specs/core/0_beacon-chain.md`](https://github.com/ethereum/eth2.0-specs/blob/master/specs/core/0_beacon-chain.md)

#### Shuffling

The shuffling follows "Swap or Not": an algorithm that can be reversed almost trivially.
It was introduced here: [`eth2.0-specs#576`](https://github.com/ethereum/eth2.0-specs/pull/576)
The Eth 2.0 spec defines the function to define a single index being shuffled [here](https://github.com/ethereum/eth2.0-specs/blob/master/specs/core/0_beacon-chain.md#compute_shuffled_index).

The implementation ideas here are:
- By reversing the rounds, you can reverse the shuffle
- For one "swap or not" decision, you only need 1 bit. I.e. 1 bit per pair in the array per round. Instead of hashing for each bit, 1 hash is used for 256 bits.
  This allows for list-wise shuffling to be ~256x faster than shuffling an individual index for each list element.

The original list-wise implementation by Vitalik (in PR 576) pre-computes all hashes of a round to do the 256-bits at a time optimization trick.
This however allocates a bulk of unnecessary hashes. As you only need 1 input buffer, and 1 output of 32 bytes at a time.
A Go implementation implementing the allocation optimization can be found here: [`protolambda/eth2-shuffle`](https://github.com/protolambda/eth2-shuffle)

Visualization of algorithm here: [`protolambda/eth2-docs#shuffling`](https://github.com/protolambda/eth2-docs#shuffling)

### BLS

BLS is still being standardized, see the Draft by the working group: https://github.com/cfrg/draft-irtf-cfrg-bls-signature

And the temporary Eth2.0 BLS spec here: [`ethereum/eth2.0-specs/specs/bls_signature.md`](https://github.com/ethereum/eth2.0-specs/blob/master/specs/bls_signature.md)

The core ideas here are:
- The aggregation of signatures lowers network bandwidth.
- The aggregation of pubkeys before verifying an aggregate signature allows for cheaper verification.
    - When the message is the same. Like attestation data. See applications by clients here:
        - [Go BLS (phoreproject, used by Prysm)](https://github.com/phoreproject/bls/pull/11)
        - [JS, Lodestar](https://github.com/ChainSafe/lodestar/pull/406)
        - [Rust Milagro, fork by Sigma Prime](https://github.com/sigp/milagro_bls/blob/33716bcdba6560f5b980f4ffae1a338c61058ee5/benches/bls381_benches.rs#L180)
- More ECC tricks are being experimented with, e.g. [experimental verify-multiple optimization by Vitalik](https://ethresear.ch/t/fast-verification-of-multiple-bls-signatures/5407) implemented [by Kirk here](https://github.com/sigp/milagro_bls/pull/6).
- Raw cryptographic BLS optimizations:
    - Less ate pairings, implemented [here by Kirk](https://github.com/sigp/milagro_bls/blob/33716bcdba6560f5b980f4ffae1a338c61058ee5/src/signature.rs#L44),
       similar to what was experimented with earlier by others like [Lovesh](https://github.com/lovesh/signature-schemes/blob/0e1d1cd0ad1cfc42131cd710256ccae19a8d91e8/bls/src/aggr_fast.rs#L74). 
    - 7 uint64 (448 bits) instead of 6 (384 bits), with 58 ("`BASEBITS`") bits used per uint64, for [ADK (paper by Miracl AMCL author)](https://eprint.iacr.org/2015/1247.pdf) optimizations.
      Including a nifty optimized montgomery modulo reduction.
     `58/64` is generally applied in all of AMCL (see [Miracl AMCL codebase](https://github.com/miracl/amcl/tree/master/version3)),
      but the `monty` trick seems to be only implemented in Rust [here](https://github.com/miracl/amcl/blob/master/version3/rust/src/big.rs#L964) (also in `sigp` fork).
    - (generative) assembly multiplication and modulo reduction functions in Go. [Phoreproject](https://github.com/phoreproject/bls/blob/master/asm/asm.go).
    - More upcoming, with BLS assembly work for Go by Cloudflare, and new Rust implementations by people from the working group.

### SSZ

Visualization of hash-tree-root and merkleization here: [`protolambda/eth2-docs -> SSZ`](https://github.com/protolambda/eth2-docs#ssz-hash-tree-root-and-merkleization)

Eth 2.0 SSZ spec here: [`ethereum/eth2.0-specs/specs/simple-serialize.md`](https://github.com/ethereum/eth2.0-specs/blob/master/specs/simple-serialize.md)

General optimizations are:
- Pre-compute (in compile time where possible) the fixed sizes and memory layout of data structures. This allows for much faster traversal of the tree, only visiting the elements during their encoding.
- SSZ is little endian (thanks Jacek), so on most CPU architectures you can copy memory from SSZ buffers to memory and back, without changing endianness.
  This is especially nice for lower-level languages like Go, Rust and Nim. And very effective for common bottom-types like uint64 lists.
- HTR/serialize instructions for static structures are static, and can be inlined/flattened during compile time. 

#### Serialization

Although SSZ does not support streaming, you can still write to and read from a stream.
For dynamic data this means you need to know the size, which means:
- for encoding: use the array-length for bottom types (most cases). Recursive length lookups may be necessary however, but are not too expensive.
- for decoding: read the first offset to determine the element count, and then allocate a typed array for it (if the length is valid).

A visit-once in-order approach would require buffering, but can be fast as well if you use pooled byte buffers (to avoid GC):
- Fixed size? -> directly encode to buffer.
- Variable size? (and non-bottom type) ->
  - get temporary buffer (reset it if it's reused),
  - write variable contents to temporary buffer
  - write offset using the length of the temporary buffer and previous written offset
  - then continue with other variable elements
  - finally, after writing all variable elements, flush the temporary buffer to the main buffer, and release it to the pool.

When pooling, if it is easy to keep separate pools per type, it helps to keep pooled buffer sizes just right (e.g. don't use a buffer big enough for a full state to write a single attestation temporarily too).

#### Hash-tree-root

Hash-tree-root is effectively just recursive merkleization. For this to work, a good merkleization base function helps a lot:

- Merge towards the root as soon as possible, instead of allocating for every branch in the tree. This reduces the memory requirements from `2*N` to `N`.
  See [here](https://github.com/ethereum/eth2.0-specs/blob/5f1cdc4acca1bb3235efdc5b63dbf9c74c4c312e/test_libs/pyspec/eth2spec/utils/merkle_minimal.py#L47) in the pyspec, also implemented in ZSSZ.
- Do not allocate a leaf chunk for each node in the tree. Instead, provide a callback to get a leaf instead of the full computed list of leaves.
  It is implemented [here](https://github.com/protolambda/zssz/blob/632f11e5e281660402bd0ac58f76090f3503def0/merkle/merkleize.go#L63) in ZSSZ, the SSZ code for ZRNT (Go-spec).
  Together with the other optimization, this reduces memory requirements to `O(log(N))`.
- Re-use the SHA-256 state machine object to create checksums with. A SIMD SHA-256 version can be nice for hashing long series of bytes (flat-hashes),
  but it may not make as much of a difference when hashing just 64 bytes at a time. Re-using one input buffer and SHA-256 state can be more effective.
- Directly provide memory slices to the hash-function as input when you can. It avoids another temporary copy. (Also works for chunks from little-endian uint64 lists).
  You will need to handle non-chunk-aligned list lengths as special case (but only for the last chunk).

#### Compile time parts

A lot of in-place and partial editing can be done when using a sufficiently expressive language.
The Nim implementation has a very advanced set of features like this, check it out if you are looking to implement SSZ for memory-constrained environments.

#### SSZ Caching

There are multiple hash-tree-root caching approaches for SSZ:

- Keep data around with the object:
    - mutable object
        - reduces necessary memory.
        - complicated to keep cache data in sync (or marked as dirty) after mutations.
    - immutable object:
        - lots of memory copies.
        - no problems with cache management.
- Keep data around aside from the object.
    - A tree of old hashes can be partially re-used when you know what bottom chunks changed.
- Use `(type, h(serialize(data)))` as key for `htr(data)`.
  Not literally, but a flat-hash in a type-specific cache can recover a hash-tree-root from the cache without collisions.
  - Works well for fetching cached hash-tree-roots almost statelessly. But the recursive approach needs care:
    storing keys for big lists is effective when there are no modifications. But when a single element changes, work has to be redone.

Note that caching serialization is not very effective, focus should be on hash-tree-root;
 an (uncached) hash-tree-root of a mainnet state is > 50x as slow as a flat-hash with serialization.

### Storage and Caching

TODO

----

## LICENSE

This text is CC0 1.0 Universal. However, note that approaches (and code in particular) in referenced links may be licensed differently.
