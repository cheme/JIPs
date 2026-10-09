# JIP-6: Compact state proofs

A compact Merkle proof for the JAM state trie. Together with the state entries it accompanies, it
proves those entries, and the absence of other keys, against a trusted state root.

## Motivation

A verifier that trusts a block's state root may need to check a small set of state values without
trusting the prover that provides them. Existing whole-node proofs, such as the CE 129 range proof,
work well for state synchronisation but are inefficient for a few unrelated keys. They ship
redundant branch data, repeat common trie paths across requests, and require rebuilding a partial
trie during verification.

The proof defined here is designed for this smaller, selective access pattern. The requested keys
and values travel beside the proof, not inside it. The proof holds only the trie structure, the
hashes of the subtrees it does not open. The verifier rebuilds every other subtree from the
requested keys and values.

One proof covers multiple keys and ranges, and proves both presence and absence. It is verified in
a single pass over a single hash stack.

Verification is done over a single iteration on content.

The proof format is independent of its transport and may be used by any protocol.

## Terms

The state trie is defined by the state Merklization appendix of the Gray Paper (GP). This
document uses the following terms for the state trie:

- The _path to a node_ is the sequence of bits walked from the root to reach it: `0` to the left
  child, `1` to the right child. The path to the root is empty, and the _depth of a node_ is the
  length of its path.
- The _hash of a node_ is the blake2b-256 hash of the node's encoding as per the GP. The hash of an
  empty subtree is the zero hash. The GP calls this hash the identity of the (sub-)trie; this
  document calls it the hash.
- A key _lies under_ a node when the path to the node is a prefix of the key, that is, when the key
  belongs to the subtree rooted at the node. The term applies to any key of 248 bits, whether or
  not it is in the state.
- _Rebuilding a subtree_ at depth $d$ from a set of (key, value) pairs means applying the GP's
  function $M$ to those pairs from bit $d$ onwards. The leaf node of a pair is the GP's $L(k, v)$.
  The recursion is at most $248 - d$ levels deep; an implementation may use an explicit stack.

This document uses the following terms for proofs:

- A _query_ is what a verifier asks a prover to prove: a set of listed keys and a set of key
  ranges. Its exact form and constraints are given under [Proving a query](#proving-a-query).
- An _entry_ is a (key, value) pair of the state. The verifier's entries are the pairs it received
  with the proof together with those it already held. Their keys are unique and ascending.
- The _claims_ of a proof are what it proves, as given under [Proof subtree](#proof-subtree). A
  query names the claims a verifier wants.

The prover holds the state. It answers a query with two things: the entries of the query, and a
proof that claims at least what the query asks for. The proof never carries the key or the value of
an entry. A listed key that is absent from the state has no entry; its absence follows from the
proof.

The verifier holds a trusted state root. It checks the entries against that root with the proof, as
defined under [Verification](#verification). This document defines the entries as an input of the
verifier, not how they are encoded. A protocol carrying the proof may let the prover omit entries
that the verifier already holds.

## Proof subtree

A proof subtree is a part of the state trie that allows rebuilding a partial tree, with both:
- elided subtree with hash, only the hash is known
- elided subtree without hash. The hash is infer from the data to be proven:
    - empty subtree if undefined
    - value node hash calculated from a partial key and an actual value
    - subtree calculated from a set of key value having a common key part (this common key 
    matches the depth of this elided content).

Note that some value hash are inlined, in this case they are seen as a standard hash for elided
subtree with hash and as a value node for elided subtree without hash.

## Encoding

The encoded proof consists of a version octet followed by four sections, with no explicit section
lengths:

```
proof              = version tags hashes raw_keys raw_values
version            = 0x00
sequences_sizes    = all size of the previous sequences.
structure          = sequence of key contents, variable size order, aligned to bits, padded by 0
structure_sizes    = sequence of key contents sizes in bits, each size is one byte and points to
                     an elided subtree without hash.
hashes             = sequence of attached elided subtree or value hashes. 32 octets per H, ordered.
```

The `version` octet identifies the encoding defined here, version 0. Later revisions of this
document may define further versions.

The `sequences_sizes` are compact size in bytes of each sequence in order.

The `structure` and `structure_sizes` sections lists all the tree path to descend into (from root
to leaf) during the verification iteration.

If content to be proven is a present single key value, we shall therefore descend up to its value node hash (248
bits minus its value node partial key length).

If content to be proven is a missing single key value, we shall descend up to the first empty node
(or not descend if the current stack is already on an empty node holding this key).

If content to be proven is a subtree (generally from a range with start key equal to end key), then (unless 
in a empty node holding the range), the depth is the depth of this range.

The tree path to unstack (going up) after a descent is define by content to be proven, for instance, if last
value or range, it will be the root (to verify root matching), between two consecutive keys it will
be the common path.

The `hashes` is a sequence of hash (aligned 32 bytes) of elided subtree that is not meant to be proven. It will be consumed
and stacked during descent and unstacked to calculate parent branch hash when going up.


Tags are packed from the most significant bits of each octet down. The first tag thus occupies bits
7 and 6 of the first octet of the `tags` section.

The `tags` section starts with one subtree open. Each `H`, `K` or `R` closes one, while each `B`
closes one and opens two more. The section ends when none remain open.

The `hashes` section holds one 32-octet hash per `H`, in tag order. If the node an `H` stands for
is a left child, its hash is encoded as the GP's branch encoding stores it: with bit 7 of octet 0
cleared.

## Verification

A verifier must report a proof whose version octet is not 0 as having an unknown version. This is a
distinct error from an invalid proof, so that a verifier can tell a later version from a corrupt
proof.

The verifier takes three inputs: the proof, a trusted state root, and its entries. While reading
the tags in order, it keeps the path to the current node and a stack of open branches, those whose
right child is still to come. Each slot of the stack is holds an optional left child's hash.
Stack is bounded to max tree depth in bits.

A verifier must reject a proof of version 0 if any of the following holds:

1. The calculated root from verification iteration do not match the expected root.
2. The set data to be verified defines data that is not under an elided subtree without hash 
3. The set of data to verify is not fully accessed during iteration.

A verifier may bound the size of the proofs it accepts, in this case the the set of data to verify
may not be fully cover, it is up to the protocol to define the subset of data that is covered
(usually by simply returning the last key or subrange), and the proof is simply covering a smaller
set that should be completed by other proofs to cover the full set. When splitting sets, there
is no idea of partial proof, this is all full proofs of consecutive set: this allows checking
root for every subset (the three previous criteria 1 can only be check after a full proof
verification).


The verification run as:

    stack = empty, path = empty, missing_elided_hash = empty_hash, target_len = 0, common_depth_up = 0, proven_root = empty_hash
    content_to_be_proven = consume first content to be proven
    loop:
        target_len = consume one bytes from structure_sizes and adds it to current length of path
        loop:
            if length of current path in bits equals to target_len
               if content_to_be_proven matches current path
                   missing_elided_hash = with current content_to_be_proven gets its hash (either partialkey and
                   value hash) or full subtree of a range hash (in this case the current stack and path should be used)
                else
                    missing_elided_hash = empty_hash
                break from loop
            else
               next_bit = consume one bit forme structure sequence
               if next_bit equals 1
                   consume one hash from hashes section and put it on stack 
               else
                   put no hash on stack (increase stack len but no content)
            content_to_be_proven, common_depth_up = consume next content to be proven, and get common depth with
            previous content to be proven (or 0 if last content). 
        loop:
            if stack length equals to common_depth_up
               break from loop
            unstack_item = pop from stack
            if unstack item is defined
                missing_elided_hash = hash of item from stack as left node and missing_elided_hash as right node
            else
                right_hash = consume one hash from hashes section
                missing_elided_hash = hash of missing_elided_hash as left node and right_hash
        if stack is empty:
            proven_root = missing_elided_hash
            break from loop
    require proven_root to match expected state root
    require every entry to be proven is consumed
    require every section is consumed exactly

## Proving a query

A query consists of:

- Listed keys: a strictly ascending sequence of state keys.
- Ranges: a sequence of pairs `[start, end]` of prefix bounds, each bound 0 to 31 octets long.

A range contains every key from `start` padded to 31 octets with `0x00` up to `end` padded to 31
octets with `0xFF`, both inclusive. `[p, p]` is thus every key starting with `p`, and `[empty,
empty]` is every key.

A query must satisfy the following, with bounds compared after padding:

- each range's `start` does not exceed its `end`;
- each range's `start` exceeds the previous range's `end`;
- no listed key lies within a range.

A prover must reject a query that violates these constraints.

From the proof perspective, a listed key is expected a value hash (hash of partial key and value) or default to an empty hash in one of its parent branches (note that if empty hash exactly matches the key, then we add it to the hash section as there is no way to know if the key  is used from the structure).

From the proof perspective, a range resolves from one to multiple subtree, that would be:

- Any subtree contained in the range with no parent subtree contain in the range.
    There for the content to be proven here would result in multiple contents. The subtree being known only during iteration over the proof, consuming next subtree here is dependant on the current path.
    No parent criteria is simply enforce by going up at first met subtree, which is defaults.

The entries of a query are the pairs of the state whose key is a listed key or lies within a range.
The prover builds the proof subtree of the query with the rules under [Construction](#construction);
its claims then include the presence of every entry, the absence of every listed key that is not in
the state, and the contents of every range.

### Construction

Construction is done over a set of contents to prove (full keys to value or partial keys to subtree (range starting
and ending at this partial key).

We iterate on state alongside content to prove and feed the require sections, quite straight forward.

## Extension to self contained proof

Query may not have all content but just a set of data to obtain alongside the proof.

### Query do not need values and knows keys.

We attach hashes with the proof.

#### node offset

- bit hash type sequence:
    - 0: an empty node
    - 1: an attached node in next sequence
- hashes sequence (all 32bytes).

### Providing values

- bit hash type sequence:
    - 0: a empty node
    - 1: a attached bit sized partial key and a value
- bit sized partial key and a value sequence (size as compact integer).

### Query example

With query being a sequence of key and ranges. Key and ranges must b deduplicated (no key in ranges), and ordered.

key should match an elided node, one and only one if there is a value attached, or an added empty node on the path.
range, from starting key to end key resolved to one elided set of node if start key equal to end key or to two elided set of node (start to common prefix, common prefix to end, with common prefix between start and end).
Again elided sets of node must match those of the proof (or have an empty node in their path).
