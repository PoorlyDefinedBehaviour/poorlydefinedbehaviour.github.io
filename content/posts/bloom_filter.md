---
title: "Bloom filter"
date: 2022-03-03T20:56:31-03:00
categories: ["today-i-learned", "probabilistic", "data-structures"]
draft: true
---

# Bloom filter

A bloom filter is a data-structure that can be used to check if a set contains an element. It uses way less memory than a [conventional set data-structure](<https://en.wikipedia.org/wiki/Set_(abstract_data_type)#Implementations>) by sacrificing accuracy.

Say we are building a [log-structured merge-tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf), we can use a bloom filter to find out if the LSM-tree contains a particular key in O(1) time in most cases, the downside is that sometimes the bloom filter would say that the LSM-tree contains a key, but it actually does not and we would go searching for the value that's mapped to the key and never actually find it.

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

# References

Network Applications of Bloom Filters: A Survey - https://www.eecs.harvard.edu/~michaelm/postscripts/im2005b.pdf
