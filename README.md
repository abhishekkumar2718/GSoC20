# GSoC 2020

## Microproject

**Consolidate test_cmp_graph_logic**

Log graph comparison logic is duplicated many times. This patch consolidates comparison and sanitation logic in lib-log-graph.

[Link to patch](https://lore.kernel.org/git/20200216134750.18947-1-abhishekkumar8222@gmail.com/)

## Proposal

**Convert mergetool to builtin**

A few git subcommands are in the form of shell and Perl scripts. This causes problems in production code - in particular, on multiple platforms.

This project rewrites git mergetool in C to improves its performance and portability. It would also lay groundwork for subsequent conversion of mergetool--lib.

[Link to mailing list discussion](https://public-inbox.org/git/CAHk66fsEjanKPtUhVnDMmU2JCL7MK+MzYbGdCAuCh00DOwgEYg@mail.gmail.com/)

[Link to proposal](https://github.com/abhishekkumar2718/GSoC20/blob/master/mergetool.md)

**Implement Generation Number v2**

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
