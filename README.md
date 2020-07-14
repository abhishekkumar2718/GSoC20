# GSoC 2020

I was selected as one of the Google Summer of Code students for Git in 2020, to work on [Implement Generation Number v2](https://summerofcode.withgoogle.com/organizations/4722049416691712/#6140278689234944). I have detailed my work in a series of blog posts [here](https://abhishekkumar2718.github.io/gsoc).

## Implement Generation Number v2

Git uses various clever methods for making operations on very large repositories
faster, from [bitmap indices for git fetch](https://githubengineering.com/counting-objects/), to [generation numbers](https://devblogs.microsoft.com/devops/supercharging-the-git-commit-graph-iii-generations/) (also known
as topological levels) in the commit-graph file for commit graph traversal
operations like git log --graph.

However, generation numbers do not always improve performance. Stolee has
previously [explored alternatives](https://lore.kernel.org/git/6367e30a-1b3a-4fe9-611b-d931f51effef@gmail.com/) for a better generation number . Backward
compatible corrected commit date was chosen because of its performance, local
computability, and backward compatibility.

[Link to mailing list discussion](https://lore.kernel.org/git/20200322093526.GA4718@Abhishek-Arch/)

[Link to proposal](https://github.com/abhishekkumar2718/GSoC20/blob/master/generation_number_v2.md)

## Patches

### Implement Generation Number v2

[Merge Request](https://github.com/gitgitgadget/git/pull/676)
[Performance Report](https://github.com/abhishekkumar2718/git/pull/1)

### Move generation, graph position into commit-slab

The struct commit is used in many contexts. However, members
`generation` and `graph_pos` are only used for commit graph related
operations and otherwise waste memory.

This wastage would have been more pronounced as we transition to
generation nuber v2, which uses 64-bit generation number instead of
current 32-bits.

[Patch](https://lore.kernel.org/git/20200604072759.19142-1-abhishekkumar8222@gmail.com/)

#### Consolidate test_cmp_graph_logic (Microproject)

Log graph comparison logic is duplicated many times. This patch consolidates comparison and sanitation logic in lib-log-graph.

[Patch](https://lore.kernel.org/git/20200216134750.18947-1-abhishekkumar8222@gmail.com/)
