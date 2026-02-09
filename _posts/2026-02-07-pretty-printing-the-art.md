---
layout: post
title: "Pretty Printing ART (Adaptive Radix Tree) in DuckDB"
date: 2026-02-09
categories:
---

In this post I'm going to give a very casual overview of the ART (adaptive radix tree)[^1] data 
structure used for indexing in DuckDB. [^2] This is mainly to give a feel of the data structure 
as a whole without any concerns for how it is integrated into the database. I haven't written in 
a while, so I thought showing a pretty printer would be a fun and easy post to write :-) I hope 
to do some deeper dives on the internals after some features and refactorings are added:

I made 
the pretty printer by prettifying some existing printing code that was in the code base, 
although it's grown into its own thing and now even has a struct, ```ToStringOptions``` 
associated with it that controls how to pretty print the tree. What started as a weekend 
project learning aid 
turned out to be really useful for quickly pinpointing bugs with large reproducers, which was 
very cool! 

Since this is a very informal post, I'm going to give a very informal description of the ART: I just think 
of an ART as a trie that saves space. It saves space vertically,
by not having multiple nodes 
for singly linked lists (just store the sequence of bytes along a path that doesn't have any 
branching), and it saves 
space horizontally,
by adaptively changing the node size 
depending on how many elements are currently being stored in the node (Nodes can store 4, 16, 48,
or 256 
children). Note this also means all operations are done in place.

Let's start with initializing an index and inserting an element into the tree:

```sql
CREATE TABLE t(s VARCHAR(5))
INSERT INTO t VALUES ('gnome')
```


```cpp
ErrorData ART::Insert(IndexLock &l, DataChunk &chunk, Vector &row_ids, IndexAppendInfo &info) {
    ...
    
    conflict_type = ARTOperator::Insert(arena, *this, tree, keys[i], 0, row_id_keys[i], GateStatus::GATE_NOT_SET,
                                        DeleteIndexInfo(info.delete_indexes), info.append_mode);
    ...
    // Print tree after every insert.
    if (tree.HasMetadata()) {
        Printer::Print(tree.ToString(*this, ToStringOptions()));
    }
```

And here is how the tree looks:

<pre style="line-height: 1.2;"> 
ART
└── Prefix: |g|n|o|m|e|0|
    Inlined Leaf [row ID: 0]
</pre>>

The underlying key type also represents varchars 
using a null terminated character, which is why they have a 0 at the end. We are also printing 
with ASCII mode, so each byte is printed as a character.

This is 
vertical compression. The root node in the tree is a prefix node, which compresses 
what would be a singly linked list in the tree. The prefix only has a pointer field for one 
child, which is currently storing the rowid of the element we just inserted. (The pointer fields 
can double up as either a pointer to a child node, or the inlined rowid of the element associated 
with 
the key we just inserted). 

Let's insert some more elements. You can see the prefix node gets split after the second letter 
once we insert 'gnaws':

<pre style="line-height: 1.2;"> 

ART
└── Prefix: |g|n|
    Node4
    ├── a
    │   └── Prefix: |w|s|0|
    │       Inlined Leaf [row ID: 1]
    └── o
        └── Prefix: |m|e|0|
            Inlined Leaf [row ID: 0]
</pre>

And if we insert some words differing on the first character the root becomes a Node4, with the 
previous tree as one of the children:

<pre style="line-height: 1.2;"> 
Node4
├── g
│   └── Prefix: |n|
│       Node4
│       ├── a
│       │   └── Prefix: |w|s|0|
│       │       Inlined Leaf [row ID: 1]
│       └── o
│           └── Prefix: |m|e|0|
│               Inlined Leaf [row ID: 0]
└── i
    └── Prefix: |g|l|o|o|0|
        Inlined Leaf [row ID: 2]
</pre>

And if we insert even more words, the Node4 grows into a Node16:

<pre style="line-height: 1.2;"> 
Node16
├── g
│   └── Prefix: |n|
│       Node4
│       ├── a
│       │   └── Prefix: |w|s|0|
│       │       Inlined Leaf [row ID: 1]
│       └── o
│           └── Prefix: |m|e|0|
│               Inlined Leaf [row ID: 0]
├── i
│   └── Prefix: |g|l|o|o|0|
│       Inlined Leaf [row ID: 2]
├── k
│   └── Prefix: |i|o|s|k|s|0|
│       Inlined Leaf [row ID: 5]
├── m
│   └── Prefix: |a|c|a|w|0|
│       Inlined Leaf [row ID: 4]
└── w
    └── Prefix: |o|r|m|s|0|
        Inlined Leaf [row ID: 3]
</pre>>

That was an example of horizontal compression. We start out with a Node4 once we have 2 elements 
to store, and grow it once we have enough children. 

Duplicates are handled by storing a nested ART tree. Let's insert 20 gnomes: 

<pre style="line-height: 1.2;"> 
Node16
├── g
│   └── Prefix: |n|
│       Node4
│       ├── a
│       │   └── Prefix: |w|s|0|
│       │       Inlined Leaf [row ID: 1]
│       └── o
│           └── Prefix: |m|e|0|
│               Gate
│               └── Prefix: |128|0|0|0|0|0|0|
│                   Node256
│                   └── Leaf |0|6|7|8|9|10|11|12|13|14|15|16|17|18|19|20|21|22|23|24|25|
├── i
│   └── Prefix: |g|l|o|o|0|
│       Inlined Leaf [row ID: 2]
├── k
│   └── Prefix: |i|o|s|k|0|
│       Inlined Leaf [row ID: 5]
├── m
│   └── Prefix: |a|c|a|w|0|
│       Inlined Leaf [row ID: 4]
└── w
    └── Prefix: |o|r|m|s|0|
        Inlined Leaf [row ID: 3]
</pre>

Nothing changes on the path along gnome, but once we get to where the rowid of the first gnome 
(0) was being stored, we store a pointer to a nested tree there instead. This is the "Gate" pointer
to the nested ART. The key space for the nested ART is just 8 byte rowids, so we can see we just 
inserted rowid's 6-25 in the nested ART. Don't mind the 128 for now, that's just part of the 
underlying key representation which is used for sorting. 

Let's insert 1000 gnomes, then delete all but 20 or so. I'm also using a setting that only 
prints the full subtrees along a certain key's path:

<pre style="line-height: 1.2;"> 
Node16
├── g
│   └── Prefix: |n|
│       Node4
│       ├── a ...
│       └── o
│           └── Prefix: |m|e|0|
│               Gate
│               └── Prefix: |128|0|0|0|0|0|
│                   Node4
│                   ├── 0
│                   │   Node7
│                   │   └── Leaf |6|37|102|198|255|
│                   ├── 1
│                   │   Node7
│                   │   └── Leaf |56|144|200|245|
│                   ├── 2
│                   │   Node7
│                   │   └── Leaf |55|111|177|199|238|
│                   └── 3
│                       Node7
│                       └── Leaf |32|66|120|153|187|231|238|
├── i ...
├── k ...
├── m ...
└── w ...
</pre>

You can see we now have some Node7's. There are some special leaf nodes for nested ART's that 
are used to just store rowid's, and thus save on some space.

This was a pretty cursory glance at the ART, I hope to share more!

---

[^1]: Leis, V., Kemper, A., & Neumann, T. (2013). *The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases*. [PDF](https://www.db.in.tum.de/~leis/papers/ART.pdf)
[^2]: Although the version I'm showing here is not yet merged, but I'm using it since it has lines, like the ```tree``` command in linux. I'm waiting to add support to print UTF-8 characters as well before merging it upstream. 