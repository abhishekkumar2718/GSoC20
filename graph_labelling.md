# Compute graph labelling for speeding up git commands

## Abstract

Git uses various clever methods for making operations on very large repositories
faster, from bitmap indices for git fetch [1], to generation numbers (also known
as topological levels) in the commit-graph file for commit graph traversal
operations like git log --graph [2].

However, generation numbers do not always improve performance. Derrick Stolee
has previously explored alternatives for a better generation number [3].
Corrected commit date was chosen because of its performance, local
computablity and backwards compatiblity. Jakub Narebski offered reachablity
intervals as another alternative for improving performance [4]. 

This project explores different alternatives and implements the best possible
choice.

[1]: https://githubengineering.com/counting-objects/
[2]: https://devblogs.microsoft.com/devops/supercharging-the-git-commit-graph-iii-generations/
[3]: https://public-inbox.org/git/86tvl0zhos.fsf@gmail.com/t/#me61383330fbb4c6b441fdfce6aec1151ecceb687
[4]: https://git.github.io/SoC-2020-Ideas/

## Goals

This project aims to:
- Explores proposed reachablity interval algorithms.
- Modify commit-graph format to store reachablity index.
- Use modified commit-graph to speed up git commands.
- If possible, teach git to update indexes without recomputing it from scratch
and make it compatible with incremental update of commit-graph file.

## Related Work

This project builds on Derrick and Jakub's terrific work on commit-graph. You
can read more about it in Derrick's posts [5].

[5]: https://devblogs.microsoft.com/devops/supercharging-the-git-commit-graph/

## Analysis of expected workload

Before we compare different reachablity algorithms, we must first understand
the expected properties of the graph and the queries [6].

I have considered the following graph properties:
1. Number of vertices
2. Edge density
3. Maximum topological level
4. R-ratio
5. Distribution of outgoing degree

| Repository | Number of vertices | Edge density | Max. topological level | R-ratio | 
|------------|--------------------|--------------|------------------------|---------|
| Linux      | 76,955,655         | 1.081        | 19,122                 |
| Git        |  5,421,355         | 1.253        | 41,382                 |
| Rails      |  6,944,095         | 1.224        | 12,264                 |
| Android    | 49,861,456         | 1.739        | 27,473                 |
| Swift      |    103,049         | 1.112        |  7,476                 |

| Outgoing degree | % of Vertices |
|-----------------|---------------|
| 1               | 71.12%        |
| 2               | 28.73%        |
| 3               | 00.04%        |
| >=4             | 00.11%        | 

On basis of the above data, we can make the following observations:
1. The size of problem is of the order of 10^7.
2. The graphs are sparse, with edge density around or less than 1.25.
3. Maximum topological level is of order 10^6.
4. Commit graphs have R-ratio, compared to usual 10^-3 to 10^-4.
5. Graphs are scale free, with outgoing degree following a very sharp power
law.

[6]: https://github.com/abhishekkumar2718/GSoC20/repo_analyzer.py

In Git, there are at least four different types of graph traversal queries with
somewhat different requirements, and affected differently by various reachablity
indexes [7].

### Pure reachability queries

A pure reachablity query is when we are interested only in answered the question
whether commit A can reach commit B or not. We are not interested in the list of
commits between A and B if any. Either a positive or negative cut index works
well. 

This query can be performed one-to-one, one-to-many, many-to-one and
many-to-many and is used by "git branch --[no-]contains", "git branch
--[no-]merged".

### Topological Ordering

Reachablity index is used to find out in-degree of considered commits (starting
from tips of each branch). Here we don't need to walk all paths, but we need to
walk all possible ways of arriving at given commit.

This query is used in "git log --topo-order" and in combination with A..B walk.

### (Po)set different or B..A range

In this type of query, we want to find out all commits reachable from commit A
that are not reachable from commit B. We need to travel (and possibly list) all
paths from A, while when travelling from B, only reachability is important.

This query is used in "git log --topo-order A..B"

### Merge base and A...B walk

We either find or all commits that are reachable from both A and B, or find all
commits reachable from A that are not reachable from B and vice versa. We walk
down one of paths for merge base and all of them for A...B if there are
multiple such commits.

This query is used in "git merge A B" and commands which accept "A...B" as a
shortcut.

There is also --ancestry-path, which displays commits that exist directly on
the ancestry chain between B and A. In other words, find all paths from B to A
if they exist. It's current implementation does not use any reachablity index.

[7]: https://public-inbox.org/git/86efc50yq5.fsf@gmail.com/

## Comparison of different indexes/algorithms

### Generation Number v2

Derrick Stolee has previously explored replacing topological levels with a
better index [8]. The indexes were compared on performance as well as
following implementation properties:

1. Compatible? - Can this index be used using same logic as minimun generation
numbers?

2. Immutable? - Are the values we store for reachablity indexes immutable?

3. Local? - Can we locally compute the value of a commit from values stored at
its parent(s)?

Based on performance, either maximun generation number or corrected commit date
should be used. Both perform comparably well but have different implementation
drawbacks.

Corrected commit was chosen as generation number v2 because:
1. Corrected commit date allows for incremental updates, which makes it removes
redundant work of recomputing index for old commits.
2. Commit-graph is not widely adopted and backwards compatiblity is not
important for now.

[8]: https://public-inbox.org/git/86tvl0zhos.fsf@gmail.com/t/#me61383330fbb4c6b441fdfce6aec1151ecceb687

### Min Post Labelling

For each commit 'v', we could store post(v), min_graph(v) which is minimun of 
post(u) for all commits reachable from 'v' and min_tree(v), which is minimun
of post(u) for subtree of a spanning tree that starts at 'v'. Then the
following conditions are true:

> if 'v' can reach 'u', then min_graph(v) <= post(u) <= post(v)

This labeling can be used to quickly find which commits are unreachable (it is
so called negative-cut filter).

> if min_tree(v) <= post(u) <= post(v), then 'v' can reach 'u'

This labeling can be used to quickly find which commits are reachable and is
called positive-cut filter.

A reachablity query of form can 'v' reach 'u' can be answered by a DFS from 'v'
pruning away unreachable paths from the decision space.

Advantages:
- Simple to implement, review and maintain.
- Requires 8n bytes of space per node.

#### GRAIL

The key ideas of GRAIL [9] are: 

1. For every (complete) subtree of T, the ordered identifiers of the nodes form
a contigous sequence of integers. Thus, the vertex set can be compactly
expressed as integer interval.

[Identifiers are assigned through a post order depth first traversal]

2. An approximate (in the sense all reachable ids are covered whereas false
positive entries are possible) interval can be used to instantly answer
negative cases. For positive cases, search continues with the expansion of the
child vertices.

Advantages:
- Great performance for negative queries.
- Requires 8 bytes of space per node.

Disadvantages:
- Poor performance for positive queries.

Taking an idea from FERRARI, if we could store an additional bit indicating
whether the interval is exact or approximate, the performance for positive
queries would be improved.

[9]: https://www.vldb.org/pvldb/vldb2010/pvldb_vol3/R24.pdf

#### FERRARI

Instead of one approximate interval as suggested by GRAIL, FERRARI [10] stores
a collection of approximate and exact intervals. This allows FERRARI to
answer negative queries and many positive queries instantly.

Based on the distribution of outgoing degrees, very few nodes will require
more than two such intervals.

Advantages:
- Better performance for positive queries than GRAIL.
- Possible to trade space for memory.

Disadvantages:
- Requires more preprocessing time than GRAIL.

[10]: https://arxiv.org/abs/1211.3375

#### PReaCH

PReaCH (Pruned Reachabilty Contraction Heirachies) [11] adapts contraction
heirarchy speedup techniques for shortest path queries to the reachablity
setting. It uses three different heuristics - RCH, Pruning based on topological
levels and Pruning based on DFS number to provide best overall performance.

Reachability Contraction Heirachy successively contracts nodes with increasing
order(v). Contract a node 'v' means removing it from graph and possibly inserting
new edges such that reachability relation in the new graph remains the same.

Any two connected nodes will have an "up-down" path between them, which reduces
the decision space tremendously. RCH and the other two heuristics [as described
in PReaCH] rely on bi-directional BFS for answering queries.

However, Commit-graph does not store children edges needed for backward
transversal. Loading the complete graph in memory and transposing it would
*defeat* the entire purpose of commit-graph. Whether we want to store children
edges is open to discussion.

Advantages:
- Best performance, both over positive and negative queries.
- Parallelizable precomputation.

Disadvantages:
- Largest preprocessing time - requires 5 DFS passes.
- Requires 8m + 64n bytes of space per commit. 

[11]: https://arxiv.org/abs/1404.4465

Note: Jakub once mentioned that PReaCH is not a viable alternative because of
it's bi-directional nature which I wholeheartedly agree with. It is not
discussed further in the proposal.

## 

## Summary

| Method                | Perf on +ve queries | Perf on -ve queries | Preprocessing time | Additional Space required | Compatible with split-graphs? |
|-----------------------|---------------------|---------------------|--------------------|---------------------------|-------------------------------|
| Corrected Commit Date | Poor                | Good                | None               | None                      | Yes                           |
| Min Post Interval     | Good                | Good                | Minimal            | 8 bytes                   | Yes                           |
| GRAIL                 | Poor                | Good                | Minimal            | 8 bytes                   | Yes                           |
| FERRARI               | Good                | Good                | Average            | 9...25 bytes              | Yes                           |

- All proposed methods are backwards incompatible with generation number v1.

## Plan

1. Community Bonding Period (April 27 - May 18)
- Implement and benchmark reachablity interval algorithms on git and linux
kernel repository.
- 

2. Move generation number to slab (May 19 - )

3. Implement

## Contributions

[Microproject] Consolidate test_cmp_graph logic
-----
Log graph comparison logic is duplicated many times. This patch consolidates
comparison and sanitation logic in lib-log-graph.

Status: Merged

Patch: https://lore.kernel.org/git/20200216134750.18947-1-abhishekkumar8222@gmail.com/

Commit: https://github.com/git/git/commit/46703057c1a0f85e24c0144b38c226c6a9ccb737

I have also reviewed patches and discussed queries with other contributors:
- https://lore.kernel.org/git/CAHk66fskrfcJ0YFDhfimVBTJZB4um7r=GdQuM8heJdZtF8D7UQ@mail.gmail.com/
- https://lore.kernel.org/git/CAHk66fvt-1RaLK8E7SDpocWM9OMAcA-gP5hjHq6r5N_FbATNgA@mail.gmail.com/
- https://github.com/git/git/pull/647#issuecomment-591978405

and others.

## About Me

I am Abhishek Kumar, a second-year CSE student at National Institute of
Technology Karnataka, India. I have a blog where I talk about my interests -
programming, fiction, and literature [].

I primarily work with C/C++ and Ruby on Rails. I am a member of my institute's
Open Source Club and student-built University Management System, _IRIS_. I have
some experience of mentoring - Creating their code style guide and being an
active reviewer [].

[]: https://abhishekkumar2718.github.io/

[]: https://iris.nitk.ac.in/about_us

## Availablity

The official GSoC coding period runs from April 27 to August 17.

My college ends on May 4 and starts for the next session on July 20.

During the break, I can easily commit to 40 hours a week and have no prior
commitments. After college begins, I can commit to around 25-30 hours.

I will be sure to update the community in case of any changes.

## Post GSoC

I would love to keep contributing to git after the GSoC period ends. There's so
much to learn from the community.

Hannes's comment on checks as a penalty that should be paid only by constant
strbufs was a perspective I had not considered [].

Interacting with Kyagi made me rethink the justifications _emphasizing commit
messages_. I was at my wit's end, which makes me appreciate my patient mentors
more and want to give back to the community.

[]: https://lore.kernel.org/git/467c035f-c7cd-01e1-e64c-2c915610de01@kdbg.org/

## Contact Information

| Name      | Abhishek Kumar                             |
| Major     | Computer Science And Engineering           |
| Institute | National Institute Of Technology Karnataka |
| E-mail    | abhishekkumar8222@gmail.com                |
| Github    | abhishekkumar2718                          |
| Timezone  | UTC+5:30 (IST)                             |

Thank you for taking the time to review my proposal!
