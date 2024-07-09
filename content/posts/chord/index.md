---
title: Chord
date: 2016-01-01
publishDate: 2016-01-01
---

I have been wanting to dive into distributed systems for a while, but this year I decided to actually do something about it.

<!--more-->

After spending quite a bit of time diving into systems papers in school, I’ve come to miss the kind of analysis and tradeoffs that reading a distributed system paper makes you consider. I’m hoping to turn this into a series of posts analyzing various systems, hopefully drawing some comparisons between them along the way. The first system to tackle is Chord[1](#fn:0), which was first published at MIT and published at SIGCOMM in 2001.

Chord is a great system to dive into for a couple of reasons. First, the service that Chord delivers can be explained in just a couple of sentences. This lets us focus more on the implementation of that contract than what the system does or does not provide. Second, the design is actually quite simple. The decentralized design gives us just a few moving parts. The intuition behind the components is also quite elegant.

Goals
-----

The Chord API is extremely simple. Our system provides a function `lookup(x)` which returns the node which stores `x`. The authors hint at ways in which you can bake replication into the system, but Chord largely ignores what data you store or how you store it.

Our goal is to provide an implementation of `lookup` with the following properties:

*   **Scalability**: Thousands or tens of thousands of nodes should be able to join the system without slowing down lookups.
*   **Fault-Tolerance**: Since our system consists of a large number of nodes, we should be able to tolerate a few nodes failing or being temporarily partitioned from the system. In practice, these failures are inevitable.
*   **Decentralization**: This is more of a design decision than a hard requirement, but our design should consist entirely of equal nodes. There’s no primary node responsible for coordinating, which leaves us without a single point of failure.

A simple, distributed hash table
--------------------------------

Our approach will eventually be somewhat sophisticated, but let’s start with the simplest possible solution. Let’s model our DHT after an ordinary hash table. Such a table normally has an array of buckets, each of which contains a number of values. When the _i_ th node joins, we will append it as the _i_ th bucket in our hash table. To minimize overhead, each node can also store a pointer only to its successor (node _i_ has a pointer to node _i + 1_) instead of an index of all nodes in the DHT. Now we can query any node we know to be in the DHT, and simply follow the successor pointers until we arrive at the node storing a given key. Keys can be hashed as they would in an ordinary hash table, where node _i_ stores all keys for which _H(k) % n == i_ for some hash function _H_.

This approach is problematic for a number of reasons, but two issues are particularly tough to solve with this approach. First, we need a consistent view of which index next to add a node at. If many nodes join and leave the DHT concurrently, this will require significant overhead. With thousands of nodes, we will need the majority to agree on which position the new node belongs at which could be quite slow. If configuration changes are sufficiently frequent as nodes come and go, we could end up wasting a lot of cycles just on configuration.

Another major issue is key ownership. Suppose, for example, that our DHT contains five nodes, each of which contains roughly an equal share of keys. What happens if node 0 goes down? It previously held all keys for which _H(k) % 5 == 0_, but after the node failure our system only contains four nodes. If we shift over all nodes (the old node 1 becomes the new node 0), a lot of keys need to move. For example, a key _k_ for which _H(k) = 6_ moves from node 1 to 2, while a key _k’_ where _H(k’) = 17_ moves from node 2 to 1. In short, almost all keys will need to move between nodes. This is clearly intractable as our system grows and nodes join and leave more frequently.

Consistent Hashing to the Rescue
--------------------------------

To solve this, we’ll need to revise our view of the world. A linear array of nodes is hard to maintain because the size of the array (and thus the domain we split keys based on) changes as nodes join and leave. We will use _consistent hashing_ to solve this. Suppose we have _K_ keys and _n_ nodes. Consistent hashing is a hashing technique where on average only _K/n_ keys need to move as _n_ changes. This is an enormous improvement over our previous linear hashing approach where roughly _K_ keys would move, especially if _n_ is very large.

In consistent hashing, our nodes become points on a circle. The size of the circle is determined by the output range of our hash function. For example, SHA-256 outputs 256 bits, so our circle will have 2^256 points. We’ll place our nodes around this circle by hashing their identifier (an IP address for example). Our five nodes from the previous example are now placed in a staggeringly huge space, but on average they should be roughly evenly spaced along the circle.

To divide keys, we’ll make each nodes responsible for all keys between its _predecessor_ and itself. When determining which node a key belongs to, we’ll use the _successor_ function which is defined on any point along the circle. Thus, _successor(x)_ is the first node we find going clockwise around the circle starting at _x_. Therefore, a key _k_ for which _H(k) = x_ (we’ll assume _H = SHA-256_) will belong to _successor(x)_.

Notice how this makes adding and removing nodes almost trivial. A new node _N_ whose identifier hashes to _y_ can run _lookup(y)_ on the DHT to get the node that will become its successor in the ring, say _M_. _N_ can contact _M_ and learn about all keys _k_ that _M_ currently owns for which _H(k) <= y_. Then, _M_ and _N_ can simply transfer keys between each other without involving any other nodes.

Not contacting any other nodes is of course a bit of a lie. In particular, the last node before _M_ along the circle will still think that _M_ is its successor, even after _N_ joins. If this nodes answers a query for a key that now belongs to _N_, it will fail to find this node. To solve this, let’s construct a linked list of the nodes around the circle. In addition to storing its successor, each node will also store its predecessor. Notes update their _predecessor_ field whenever a new node notifies them that they have joined before them. We’ll use the following pseudocode:

    def notify(new_node)
      if self.predecessor.nil? ||
        (self.predecessor.index...self.index).include?(new_node.index)
        self.predecessor = new_node
      end
    end


Nodes then periodically run a `stabilize` function, which uses this field to check that their view of the circle is consistent:

    def stabilize
      x = self.successor.predecessor
      if (self.index...self.successor.index).include?(x)
        self.successor = x
      end
    end


Now we have a pretty useful foundation up and running. Nodes can come and go, and they’ll only need to pull keys from the node whose keys they are taking. Nodes also periodically refresh their view of the world to make sure that successor pointers are always correct. The _successor_ pointers are therefore used both when adding nodes and keys, which is quite elegant. Unfortunately, this linked list of nodes has a significant drawback: If we happen to start our search for a key or addition of a node far away from where we will eventually end up, we will need to take many slow steps along the circle first. With 10000 nodes in our DHT, we can expect to take 5000 steps on average to find the point we’re interested in. Even if we could somehow get to 1ms each time we contact a node, we’ll still spend five seconds walking the circle, which is clearly unacceptable.

Finger Tables to the Rescue
---------------------------

To see how we can speed this up, let’s take a step back. At this point, our nodes form a circular, ordered linked list, a data structure not particularly well-suited for efficient lookups. A skip list, however, is essentially also a linked list, but with additional “fast track” indexing built into the data structure. If you’ve never seen them, I recommend checking them out (although they won’t be critical for understanding Chord) [2](#fn:1).

Finger tables provide a similar indexing structure in Chord. Instead of a central index, each node will maintain an index that helps skip past successors. The finger table for a node _n_ is simply a list of node identifiers. The _i_ th identifier is the result of the query `successor(n + 2^(i - 1))`. Note that the first identifier is the query `successor(n + 1)` which we’ve previously seen as the successor of _n_. Instead of walking around the circle node by node, we can consult the finger table and make the longest possible jump when answering a query. If our node has identifier 2, for example, and receive the query `lookup(70)`, we can jump to the 6th node in the finger table. That node could be the owner of 70 (if no nodes have identifiers between 66 and 69), but even in the worst case we’ll make significant progress. Even if our circle is saturated with nodes, we’ll jump from 2 to 66 and then from 66 to 70, just two hops. That is remarkable compared to the 68 hops we would need without the finger table.

Since finger table entry offsets are successive powers of two, an up-to-date table will always help us jump past half the nodes between us and the target node. To see this, I find it helpful to think about the _i + 1_ th node in the finger table. In the worst case, is the immediate successor of the target node. In other words, we are approximately _2^(i + 2)_ steps along the circle away. By construction, the _i_ th node must be at least as close to the target node as it is to us. This must be true since it is the successor of the point halfway between us and the target (`successor(n + 2^(i + 1))`, more precisely). Following the \_i\_th finger therefore cuts the distance between us and the target node in half.

You may have noticed that I have stressed the need for an _up-to-date_ finger table above. It turns out that finger tables can get stale quite quickly unless we are dilligent about updating them. This is fairly obvious since our usual update mechanisms only involved a new node’s immediate successor and predecessor. We have no “backwards” pointers in a finger table, so nodes do not know which finger tables they are part of. Therefore we do not have a perfect way to keep finger tables in sync. The Chord paper solves this using the following helpers:

    def update_others
      (1..m).each do |i|
        p = find_predecessor(self.index - 2 ** (i - 1))
        p.update_finger_table(self, i)
      end
    end

    def update_finger_table(node, i)
      if (self.index..self.finger[i].index).include?(node.index)
        finger[i] = node
        p = self.predecessor
        p.update_finger_table(node, i)
      end
    end


In the code above, `find_predecessor` returns the node immediately preceding a given index. The distance between this node and our node must be at least `2 ** (i - 1)`, so our node could be its _i_ th finger. We may also be the _i_ th finger of some number of predecessors of this node, which is why we keep following predecessor links.

Virtual Reality
---------------

As described above, the Chord protocol works well under one important assumption: That nodes each own a relatively equal number of keys. The protocol starts breaking down for a number of reasons if that assumption is violated. Any node with a disproportionate number of keys could easily become a bottleneck. A client that always contacts the same node for queries could also easily get unlucky if this crowded node is far away along the circle.

Chord solves this by introducing the concept of “virtual nodes”. Each node in the circle hosts a number of virtual nodes, each of which is assigned a random identifier which places it a random point around the circle. With virtual nodes, a physical node now needs to be responsible for several popular virtual nodes before it experiences increased load. I will not spend too much time on the probability argument here since this blog post from eighty-twenty does an excellent job[3](#fn:2)

I will note, however, that there is an obvious tension between scalability and overhead when we use virtual nodes. With _r_ virtual nodes per physical nodes, we will need _r_ times as much storage in our routing tables. Where finger tables normally contained _log(M)_ entries, they will now need to hold _r log(M)_ pointers. The Chord authors suggest using _r = log(N)_, which would require _log(N) log(M)_ storage. Even with _10^6_ nodes, however, _r = log\_2(10^6) = 20_, which is still quite manageable.

Curiously, adding more nodes does not affect expected query path length as long as _r = log(N)_. to see this, remember that queries are logarithmic in the number of nodes, which previously gave us _O(log N)_ query path lengths. As the number of nodes increases by _r_ we get an expected path length of _O(log(r N)) = O(log (N log N)) = O(log N)_.

1.  [Chord: A Scalable Peer-to-peer Lookup Service for Internet Applications](https://pdos.csail.mit.edu/papers/chord:sigcomm01/chord_sigcomm.pdf) [↩](#fnref:0)

2.  [Skip Lists: A Probabilistic Alternative to Balanced Trees](http://epaperpress.com/sortsearch/download/skiplist.pdf) [↩](#fnref:1)

3.  [Chord-style DHTs don’t fairly balance their load without a fight](http://eighty-twenty.org/2013/03/26/span-lengths-in-chord-style-dhts.html) [↩](#fnref:2)
