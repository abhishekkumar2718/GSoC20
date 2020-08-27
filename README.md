# GSoC 2020

I was selected as one of the Google Summer of Code students for Git in 2020 to work on [Implement Generation Number v2](https://summerofcode.withgoogle.com/organizations/4722049416691712/#6140278689234944). I have detailed my work in a series of blog posts [here](https://abhishekkumar2718.github.io/gsoc).

## Implement Generation Number v2

Git uses various clever methods for making operations on very large repositories
faster, from [bitmap indices for git fetch](https://githubengineering.com/counting-objects/), to [generation numbers](https://devblogs.microsoft.com/devops/supercharging-the-git-commit-graph-iii-generations/) (also known
as topological levels) in the commit-graph file for commit-graph traversal
operations like git log --graph.

However, generation numbers do not always improve performance. Stolee has
previously [explored alternatives](https://lore.kernel.org/git/6367e30a-1b3a-4fe9-611b-d931f51effef@gmail.com/) for a better generation number . Backward
compatible corrected commit date was chosen because of its performance, local
computability, and backward compatibility.

[Link to mailing list discussion](https://lore.kernel.org/git/20200322093526.GA4718@Abhishek-Arch/)

[Link to the proposal](https://github.com/abhishekkumar2718/GSoC20/blob/master/generation_number_v2.md)

## Patches

### Implement Generation Number v2

This patch series implements the corrected commit date offsets as generation number v2, along with other pre-requisites.

Git uses topological levels in the commit-graph file for commit-graph traversal operations like git log --graph. Unfortunately, using topological levels can result in a worse performance than without them when compared with committer date as a heuristics. For example, git merge-base v4.8 v4.9 on the Linux repository walks 635,579 commits using topological levels and walks 167,468 using committer date.

Thus, the need for generation number v2 was born. New generation number needed to provide good performance, increment updates, and backward compatibility. Due to an unfortunate problem [1], we also needed a way to distinguish between the old and new generation number without incrementing graph version.

[1]: https://public-inbox.org/git/87a7gdspo4.fsf@evledraar.gmail.com/

Various candidates were examined (https://github.com/derrickstolee/gen-test, abhishekkumar2718#1). The proposed generation number v2, Corrected Commit Date with Mononotically Increasing Offsets performed much worse than committer date (506,577 vs. 167,468 commits walked for git merge-base v4.8 v4.9) and was dropped.

Using Generation Data chunk (GDAT) relieves the requirement of backward compatibility as we would continue to store topological levels in Commit Data (CDAT) chunk. Thus, Corrected Commit Date was chosen as generation number v2. The Corrected Commit Date is defined as:

For a commit C, let its corrected commit date be the maximum of the commit date of C and the corrected commit dates of its parents plus 1. Then corrected commit date offset is the difference between corrected commit date of C and commit date of C.

We will introduce an additional commit-graph chunk, Generation Data chunk, and store corrected commit date offsets in GDAT chunk while storing topological levels in CDAT chunk. The old versions of Git would ignore GDAT chunk, using topological levels from CDAT chunk. In contrast, new versions of Git would use corrected commit dates, falling back to topological level if the generation data chunk is absent in the commit-graph file.

For mixed generation number environment (for example new Git on the command line, old Git used by GUI client), we can encounter a mixed-chain commit-graph (a commit-graph chain where some of split commit-graph files have GDAT chunk and others do not). As backward compatibility is one of the goals, we can define the following behavior:

While reading a mixed-chain commit-graph version, we fall back on topological levels as corrected commit dates and topological levels cannot be compared directly.

While writing on top of a split commit-graph, we check if the tip of the chain has a GDAT chunk. If it does, we append to the chain, writing GDAT chunk. Thus, we guarantee if the topmost split commit-graph file has a GDAT chunk, rest of the chain does too.

If the topmost split commit-graph file does not have a GDAT chunk (meaning it has been appended by the old Git), we write without GDAT chunk. We do write a GDAT chunk when the existing chain does not have GDAT chunk - when we are writing to the commit-graph chain with the 'replace' strategy.

[Merge Request](https://github.com/gitgitgadget/git/pull/676)

[Link to mailing list discussion](https://lore.kernel.org/git/pull.676.v2.git.1596941624.gitgitgadget@gmail.com/T/#meefa4ee2a1cfab06fe760e1a9e596a2dc8acdef8)

### Performance of Metadata, Generation Data

As old versions of Git can `die()` when they encounter a commit graph with different graph version, our implementation of generation number v2 has to be backwards compatible. I wrote up an investigatory report comparing different possible approaches, following [Generation Number v2], a similar report by Dr. Stolee.

The following approaches were implemented:

- Metadata/Versioning Chunk: Let's introduce an optional metadata chunk to store generation number version and store corrected date offsets. Since the offsets are backward compatible, Old Git would still yield correct results by assuming the offsets to create corrected commit dates.

- Generation Data Chunk: We could move the generation number v2 into a separate chunk, storing generation number v1 (topological levels) in CDAT, and the corrected commit dates into a new "GDAT" chunk. Thus, old Git would use generation number v1, and new Git would use corrected commit dates from GDAT.

We also considered two possible definitions for generation number v2:

- Corrected Committer Dates (as described above).

- Corrected Committer Dates with Monotonically Increasing Offsets (generation number v5) defined with the following conditions:

```
1. committer_date(C) + offset(C) > commiter_date(P) + offset(P)
2. offset(C) > offset(P)
```

However, it was found that generation number v5 did not perform as well as generation number v2 and was eventually scrapped. We chose to implement Corrected Committer Date using Generation Data chunk.

### Move generation, graph position into commit-slab (merged)

The struct commit is used in many contexts. However, members
`generation` and `graph_pos` are only used for commit graph related
operations and otherwise waste memory.

This wastage would have been more pronounced as we transition to
generation nuber v2, which uses 64-bit generation number instead of
current 32-bits.

While the overall test suite runs as fast as master
(series: 26m48s, master: 27m34s, faster by 2.87%), certain commands
like `git merge-base --is-ancestor` were slowed by 40% as discovered
by Szeder Gábor [1]. After minimizing commit-slab access, the slow down
persists but is closer to 20%.

Derrick Stolee believes the slow down is attributable to the underlying
algorithm rather than the slowness of commit-slab access [2] and we will
follow-up in a later series.

[Link to mailing list discussion](https://lore.kernel.org/git/20200604072759.19142-1-abhishekkumar8222@gmail.com/)

### Consolidate test_cmp_graph_logic (Microproject, merged)

Log graph comparision logic is duplicated many times in:
- t3430-rebase-merges.sh
- t4202-log.sh
- t4214-log-graph-octopus.sh
- t4215-log-skewed-merges.sh

This patch consolidates comparision and sanitization logic in
lib-log-graph.

Replaces duplicated code with local and lib helpers - making test
scripts cleaner and more readable.

Closes gitgitgadget issue #471.

[Link to mailing list discussion](https://lore.kernel.org/git/20200216134750.18947-1-abhishekkumar8222@gmail.com/)
