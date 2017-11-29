---
layout: post
title: "Efficient HAMT Implementations"
description: ""
category: 
tags: []
---

This post focuses on the efficient implementations of Hash Array Mapped Tries (HAMT), an immutable persistent data structure, in the JVM. For an introduction to immutable persistent data strucutres, have a look at [this post]({{ site.url }}/2017/09/immutable-persistent-data-structures) or resort to Wikipedia. Most of this post is based on [this paper](https://michael.steindorfer.name/publications/oopsla15.pdf). As a short recap. Persistent data structures preserve previous states after update operations, hence the name persistent. This property makes data structures effectively immutable and thus very well suited in concurrent environmens. The usage of a trie structure enables structural sharing, i.e. large parts of the data structure can be be safely shared between different versions.  
[Short HAMT Recap?]

# Compact vs. Expanded Representation
The principle design choice in the implementation of a HAMT is whether to store values in internal nodes of the trie as was suggested in the original design by Phil Bagwell[^1], or in leaf nodes only. Consider an example with the following three objects.
```scala
hash("Simba") = 52 = 0b110100
hash("Timon") = 18 = 0b010010
hash("Pumba") = 10 = 0b001010
```
The following image shows how the HAMT is constructed when values are stored in leaf nodes only (a) as compared to the compact representation where values can also be stored in internal nodes (b).
![HAMT Layouts]({{ site.url }}/assets/2017_hamt/hamt_layout.png)
The compact representation (b) is more efficient in terms of memory due to the absence of the leaf nodes which results in a significantly lower memory footprint. On top of that it also increases memory locality because there are less levels of indirection by omitting the intermediary leaf nodes.  
The implementation where values are stored in leaf nodes separate from inner prefix nodes (a) has it's advantages too. The leafs allow for additional information to be stored along with the actual elements. Scala, for example, memorises the hash codes of the elements inside the leafs. This allows for several runtime optiminisations, namely fast failing on negative lookups and avoiding recalculation of hash codes up prefix expansion.
Let's say "Rafiki" is retrieved, where `hash("Rafiki") = 44 = 0b101100`. The two least significant bits `00` are taken and then, in

case (a):
1. Traverse to position `00` in the root array ==> leaf node
2. Compare with stored hash in leaf
3. no match

case (b):
1. Traverse to position `00` in the root array ==> value
2. Calculate hash code of value ==> `hash("Simba") = 52 = 0b110100`
3. Compare hash codes
4. no match

The same happens if "Rafiki" is inserted. The prefix at position `00` of the root node needs to be expanded in order to make space for both objects. In case (b), the hash code of "Simba" needs to be recalculated to know where to store the object in the new array whereas in case (a) the hash code is already known and the object can be inserted straight away. Calculating hash codes is potentially very expensive and the time it takes to recalculate those can be traded for higher memory usage.  
If memory usage is the primary concern, the compact implementation (b) is the better choice whereas the expaned design (a) should be chosen when the runtime of operations is more important. Intrestingly, the designers of Clojure have chosen the former and of Scala the latter approach in their default collection implementations[^2].

[^1]: Phil Bagwell, *Ideal Hash Trees* <https://idea.popcount.org/2012-07-25-introduction-to-hamt/idealhashtrees.pdf>
[^2]: As of version 2.12 and 1.8 for Scala and Clojure, respectively.

```java
abstract class HAMT<K,V> {
	class HamtNode{
		int bitmap;
		Object[] contentArray;
	}
}
```

```java
abstract class HAMT<K,V> {
	class HamtNode{
		int bitmap;
		Object[] contentArray;
	}
	class HamtLeaf{
		String hashCode;
		V value;
	}
}
```

So far the implementations are sufficient to implement a HAMT-based set. How do the two major designs described so far translate to a map implementation?

# HAMT-based Maps

Implementing a map with the expanded design where all values are in leaf nodes requires no major changes. All that has to be added is a field containing the value assiciated with the key.

```java
abstract class HAMT<K,V> {
	class HamtNode{
		int bitmap;
		Object[] contentArray;
	}
	class HamtLeaf{
		String hashCode;
		K key;
		V value;
	}
}
```

Map implementation with compact representation are slightly more tricky. The intermediary nodes need a place to store the values associated with the keys. One way to achieve that is to double the size of the content array in the nodes.


# References