# data_structures

Data structures with a formal-verification counterpart in Lean 4.

## What it provides

The public entry point is root module `wavl_trees.nim`, which re-exports
the implementation under `src/`. It is imported by path.

| import                                 | provides                                       |
| -------------------------------------- | ---------------------------------------------- |
| `workspace/data_structures/wavl_trees` | WAVL tree used as a longest-prefix-match index |

### WAVL (Weak AVL) tree — `wavl_trees.nim`

An intrusive, index-based, `seq`-backed WAVL tree.
Self-balancing BST with rank differences of 1 or 2 between parent and child,
giving `O(log N)` operations with amortized `O(1)` restructuring per insert/delete.

- **Intrusive design**
  nodes are not separately allocated, each entry is an index into a parallel
  `WavlLink` `seq` (`p`/`l`/`r`/`rank`) living alongside the caller's data.
- **Zero tree-node GC allocations**
  contiguous and cache-friendly, 200K nodes ≈ 3.2 MB of links per the header notes.
- **Removal dance**
  integrates with Nim `seq.del` swap-pop, `fixLinksAfterIndexRemap` updates
  only the ≤3 affected references in `O(1)`.

API takes `wavlInit`, `wavlInsert`, `wavlFind`, `wavlMin`, `wavlMax`, `wavlDelete`,
`fixLinksAfterIndexRemap`, and the `wavlFindBestMatch` template.

### Longest-prefix-match via signed comparator

`wavlFindBestMatch` uses a comparator returning the signed position where
the compared keys first diverge rather than just `-1/0/+1`.
The sign drives BST navigation while the magnitude is the shared-prefix length.

- **Use case**
  the KV cache radix trie, keyed by 256-token pages, indexes pages
  through this longest-prefix match.
- **On a miss**
  the neighbor with the longest shared prefix is returned in pure
  `O(log N)`, no linear scan.

## Formal verification

- Lean 4 formalization of the Nim implementation
  at [`src/wavl_trees.lean`](src/wavl_trees.lean),
  after Haeupler/Sen/Tarjan 2015 and Gillon 2024.

## Tests

- `tests/test_wavl_trees.nim`.

## Status

WAVL tree with LPM support and its Lean formalization are implemented.
The snapshot ships only the structures the transformers stack uses.
Additional data structures live in the upstream project.

## Related

- Root project, [`../../README.md`](../../README.md)
