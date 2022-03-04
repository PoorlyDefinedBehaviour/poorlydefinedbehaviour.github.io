---
title: "Bloom filter"
date: 2022-03-03T20:56:31-03:00
categories: ["today-i-learned", "probabilistic", "data-structures"]
draft: false
---

# What's a Bloom filter

A bloom filter is a data-structure that can be used to check if a set contains an element. It uses way less memory than a [conventional set data-structure](<https://en.wikipedia.org/wiki/Set_(abstract_data_type)#Implementations>) by sacrificing accuracy.

## Example

Say we are building a [log-structured merge-tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf), we can use a bloom filter to find out if the LSM-tree contains a particular key in O(1) time in most cases, the downside is that sometimes the bloom filter would say that the LSM-tree contains a key, but it actually does not and we would go searching for the value that's mapped to the key and never actually find it.

It is used in a lot of [places](https://en.wikipedia.org/wiki/Bloom_filter#Examples).

# How it works

A bloom filter is just a [bit-set](https://en.wikipedia.org/wiki/Bit_array) that uses `n` [deterministic hash functions](https://en.wikipedia.org/wiki/Hash_function#Deterministic) to add elements to it.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156580660-d95e19fa-8d63-40e3-9fda-fc27f3eea311.png" />
</p>
<p align="center"><i>empty bit-set</i></p>

## Adding elements to the set

To add the key `bob` to the set, we run the key through each of the `n` hash functions and map the hash function output to one of the positions in the bit-set and for each position, we flip the bit to `1`.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156581183-7a853828-d0fc-4024-b3d0-22fe9cbbdbe6.png" />
</p>
<p align="center">
  <i>bit-set after bob was added to the bloom filter</i>
</p>

## Finding out if the set contains an element

To find out if the set contains the key `bob`, we run the key through each of the `n` hash functions again -- since the hash functions must be deterministic they will **always** map to the same position in the bit-set -- and check if the bit is set to `1` for each of the bit-set positions we reached after running the key through the hash functions. If every hash function maps to a bit set to `1`, it means the key is in the set.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156583072-424f00f0-3eda-48ad-96c7-5a460beca424.png" />
</p>
<p align="center">
  <i>bob is in the set because every hash function mapped it to a bit set to 1</i>
</p>

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156583592-e96d831c-21be-420a-ae10-2dde7f8a0cfe.png" />
</p>
<p align="center">
  <i>alice is not in the set because not every hash function mapped to a bit set to 1</i>
</p>

## False positives

Since [collisions](https://en.wikipedia.org/wiki/Hash_collision) can happen some keys will be mapped to bits that were set to `1` when other keys were added to the set. In this case, the bloom filter will say that it contains the key even though it does not.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156584354-9b605741-0c11-4e1c-bf01-e9e5b61453d4.png" />
</p>
<p align="center">
  <i>the bit-set after bob was added to it</i>
</p>

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156584791-cbde88f8-c13b-40e8-b568-e21cc3b82c86.png"/>
</p>
<p align="center">
  <i>since john maps to the same bits as bob and the bits were set to 1 after bob was added to the set, we got a false positive</i>
</p>

## Removing an element from the set

As it stands, removing an element from the set is not actually possible. If we had a bloom filter that uses 3 hash functions that looks like this after adding `alice` and `bob` to it:

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156785419-e7501222-9e4f-4a59-959c-5940c3e892d8.png" />
</p>
<p align="center">
  <i>bloom filter after adding alice and bob to it with 3 hash functions</i>
</p>

Note that `alice` and `bob` hash to the same position in the bit-set for some of the hash functions

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156785275-dc3016f2-211a-4b17-ac7e-4c74ca8c93ae.png" />
</p>
<p align="center">
  <i>bits shared between alice and bob are in white</i>
</p>

The naive solution is to remove `alice` from the bloom filter by setting the bits mapped by hash<sub>i</sub>(alice) to `0`:

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156785668-069767c2-f08e-4c72-a8c0-476bef9b92be.png" />
</p>
<p align="center">
  <i>bits that were flipped to 0 are in white</i>
</p>

Now, let's check if `alice` is in the set, for a key to be in the set hash<sub>i</sub>(key) must map to bits set to `1`

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156786462-6413cea3-c222-422e-b4fb-fbddcc8ce0c6.png" />
<p/>
<p align="center">
  <i>hash<sub>i</sub>(alice) maps to bits set to 0 which means alice is not in the set</i>
</p>

`alice` is not in the set as expected. Let's see if `bob` is still in the set, it should be since we didn't remove it.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156786853-65f9841c-a8f6-41be-a024-2492aca321e9.png" />
</p>
<p align="center">
  <i>not every hash<sub>i</sub>(bob) maps to bits set to 1 which means bob is not in the set, bits set to 0 after removing alice from the set are in white</i>
</p>

`bob` is not in the set anymore, even though we didn't remove it. The problem is that since keys may share the positions in the bit-set, we cannot just flip bits back to `0` to remove a key from the set because in doing so we may flip bits that are used by other keys.

## Counting bloom filter

Since we cannot flip bits back to `0` to remove a key from the set, we could maintain a counter instead of a single bit. When a key is added to the set, the counter is incremented and when a key is removed from the set, the counter is decremented. If the counter reaches 0, it means no keys are mapped to the position.

Positions that have more than one key mapped to it will have a counter greater than `1`.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156789341-ecabd5c1-fdb5-4343-8622-685684cca910.png" />
</p>
<p align="center">
  <i>positions in white are shared between two or more keys and have a counter greater than 1</i>
</p>

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/156789826-1b0bd61c-69ef-4da7-8918-65d3bae15a4b.png" />
</p>
<p align="center">
  <i>removing alice from the set by decrementing the counters mapped by hash<sub>i</sub>(alice)</i>
</p>

After decrementing the counters, not every hash<sub>i</sub>(alice) maps to a counter greater than `0` which means `alice` is not in the set anymore. Unlike the bloom filter that uses only bits, hash<sub>i</sub>(bob) still maps to counters that are greater than `0` which means `bob` is still in the set.

## Example in Rust

https://github.com/PoorlyDefinedBehaviour/bloom_filter/

# References

Network Applications of Bloom Filters: A Survey - https://www.eecs.harvard.edu/~michaelm/postscripts/im2005b.pdf  
Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems - Martin Kleppmann  
