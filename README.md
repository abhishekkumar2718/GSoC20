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
