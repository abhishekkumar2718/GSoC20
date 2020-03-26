# Implement Generation Number v2

## Abstract

Git uses various clever methods for making operations on very large repositories
faster, from bitmap indices for git fetch [1], to generation numbers (also known
as topological levels) in the commit-graph file for commit graph traversal
operations like git log --graph [2].

However, generation numbers do not always improve performance. Stolee has
previously explored alternatives for a better generation number [3]. Backward
compatible corrected commit date was chosen because of its performance, local
computability, and backward compatibility.

[1]: https://githubengineering.com/counting-objects/
[2]: https://devblogs.microsoft.com/devops/supercharging-the-git-commit-graph-iii-generations/
[3]: https://lore.kernel.org/git/6367e30a-1b3a-4fe9-611b-d931f51effef@gmail.com/

## Goals

This project aims to:
- Move `generation`, `graph_pos` members of struct commit into a commit slab.
- Examine methods to distinguish commit-graph with legacy generation number and
generation number v2.
- Update generation number to v2, modifying all places that reference or
compute generation numbers.
- Use generation numbers to speed up implementation of `--ancestry-path` and
possibly other commands.
- Extend Stolee`s performance test suite.
- Examine performance improvements.

## Overview

Git walks the commit graph for many reasons, including listing and filtering
commit history and computing merge bases.

These operations can become slow as the commit count grows. The two main costs
here are decompressing and parsing commits and walking the entire graphs to
satisfy topological order constraints.

Stolee introduced commit-graph file to accelerate this process. The commit-graph
file stores commit graph structure along extra metadata to speed up graph walks.
A chunk includes commit OID, list of parents, commit date, root tree OID and 
generation number.

Generation number v1 was defined as one more than the length of the longest path
from A to the root commit. Then,

    If A and B are commits with generation numbers N and M, and N <= M, then
    A cannot reach B.

This property reduces the time to walk commits and determine topological
relationships significantly.

### Performance Regression

While testing git v2.19.0-rc1, Stolee came across a performance regression due
to using topological levels [4]. The discussion can be summarized as:

Topological level changes the order in which the commits are inspected and
gives a performance regression if it chooses to walk a longer path than
compared to date-based heuristic.

For example, consider the performance before and after using topological levels
on `git merge-base v4.8 v4.9`:

  before topological levels: 0.122s
   after topological levels: 0.547s


The topology of this can be described as:
```
   v4.9
    |  \
    |   \
   v4.8  \
    | \   \
    |  \   |
   ...  A  B
    |  /  /
    | /  /
    |/__/
    C
```

Here, the "..." means a "very long line of commits". A and B have generation
one more than C.

If we order commits by generation, then commit date, B is removed from the queue
only after many commits reachable from v4.8 have been explored.  However, if
ordering by commit date removes A and B from the queue early and C is inserted.
As C is processed, the rest of queue is marked as STALE and loop terminates.

[4]: https://lore.kernel.org/git/pull.28.git.gitgitgadget@gmail.com/

### Alternatives to Topological Level

To improve generation numbers, Stolee investigated other alternative
definitions for generation numbers. The extended exploration and report are
available here [5].

Corrected commit date was chosen as the next generation number by overall
consensus because of its performance, local computability, and immutability.

However, it was not backward compatible with current implementation i.e., older
clients do not report correct values if they ignore reachability index version.
Existing clients would fail altogether with commit-graph v2 instead of
switching commit-graph feature off, as discovered by Ævar Arnfjörð Bjarmason [6].

[5]: https://lore.kernel.org/git/6367e30a-1b3a-4fe9-611b-d931f51effef@gmail.com/
[6]: https://lore.kernel.org/git/87a7gdspo4.fsf@evledraar.gmail.com/

### Corrected Commit Date With Strictly Monotonic Offset

Corrected commit date was later modified to _Corrected Commit Date With
Strictly Monotonic Offset_, defined as follows [7]:

For a commit C, let its _offset commit date_ (denoted by odate(C))
be a commit date plus some offset, i.e. odate(C) = date(C) + offset(C),
such that:

1. Offset commit date is greater than the maximum of the commit date of
   C and the offset commit dates of its parents.

    If odate(A) < odate(B), then A cannot reach B.

2. The offset of a commit is one more than the maximum offset of a parent, or
   more.

    If offset(A) < offset(B), then A cannot reach B.

Since the offset are strictly greater than the offset of a parent, the old
clients give correct values for the odate as well. `git commit-graph verify`
would fail since the offsets are not generation numbers in all cases.

This is an acceptable trade off because users can re-write the commit graph
after verify fails.
 
A rough implementation by Jakub can be read here [8].

[7]: https://lore.kernel.org/git/86o8ziatb2.fsf_-_@gmail.com/
[8]: https://github.com/derrickstolee/git/pull/10

## Distinguishing Generation Numbers

As explained above, commit-graph format version cannot be used to distinguish
between generation numbers. We want new Git to recognize if commit-graph uses
generation v1, use those generation numbers correctly and update commit-graph
file. The solutions suggested are:

- metadata / versioning chunk: Introduce an optional chunk to store generation
number as metadata and store offsets in CDAT. For a chain of commit-graph files
with both versions, best choice would be flatten the chain and recompute values
from scratch.

- new chunk for generation number: Introduce an optional chunk to store the
offset. Fill in CDAT with either generation zero or compute topological level.
This removes the restriction of storing generation number in CDAT. However,
since Git lazily parses commit data, seperating generation number from CDAT
would increase time spent in I/O.

Metadata chunk seems like the better solution for corrected commit date.
However, generation number chunk should be revisited if later versions of
generation numbers require more than 30 bits.

## Plan

### Community bonding period (May 4 - June 1)

1. Extend Stolee`s performance test suite.

To investigate replacements for generation numbers, Stolee implemented
prototypes and compared performance on a range of queries.

It currently tests:
- `git log --topo-order -N`: measures commits walked to determine all possible
ways of arriving at given commit.
- `git log --topo-order -10 A..B`: measures commits walked to determine commits
reachable from B but not from A.
- `git merge-base A B`: measures commits walked to find the lowest (closest)
ancestor for both A and B.

We can also add tests for:
- `git branch --[no-]contains A`: measures commits walked to determine if A is
reachable from the tip of a given branch for all branches (many to one query).
- `git branch --[no-]merged A`: measures commits walked to determine if the tip
of a given branch is reachable from A for all branches (one to many query).
- `git push/fetch`: measures commits walked to compute HAVES/WANTS of a client
(many to many query) [9].

2. Find other operations that can be optimized using generation numbers.

As Jakub pointed out in the discussion on generation numbers, --ancestry-path
can be optimized using generation numbers [10]. Discuss and find any other
operation which could use generation numbers.

3. Explore commit, commit-graph, commit-reach, revision

[9]: https://github.blog/2015-09-22-counting-objects/
[10]: https://lore.kernel.org/git/861s82wdbp.fsf@gmail.com/

### Move `generation`, `graph_pos` into a slab (June 1 - June 15, 2 weeks)

struct commit is used in many contexts. However, the struct members
`generation` and `graph_pos` are only used for commit-graph related operations
and otherwise waste memory.

Using a commit slab reduces space of an individual commit object, which becomes
crucial as we load thousands of them. We can dynamically associate commits
with their generation/graph positions when needed.

Commit slab for `generation` is 64 bits wide as it stores odate instead
of offset [11]. Commit slab for `graph_pos` is simpler, with a straightforward
allocation of 32 bits.

Note: Commit-graph chunks would still store offset and will be used to compute
corrected date on initialization.

[11]: https://lore.kernel.org/git/cfa2c367-5cd7-add5-0293-caa75b103f34@gmail.com/

### Implement backward compatible corrected commit date (June 15 - August 2, 7 weeks)

1. Implement metadata chunk (June 15 - June 29, 2 week)

2. Implement backward compatible corrected commit date (June 29 - July 13, 2 weeks)

Topological levels are computed using a DFS in `compute_generation_numbers` of
commit-graph.c. The condition

  generation(C) = max(generation(P)) + 1,
  where C is a commit and P are its parent(s).

needs to be redefined as:

  generation(C) = max(max(generation(P)), correction(C)) + 1,
  where correction(C) is max(cdate(P) - cdate(C), 0). 

Definitions of `GENERATION_NUMBER_{INFINITY, MAX, ZERO}` remain the same.

The functions `compare_commits_by_gen_then_commit_date` and
`compare_commits_by_commit_date` can be replaced by `compare_commits_by_gen` as
all three *should* return the same order.

--> First evaluation (June 29 - July 3)

--> End Semester Exams (June 29 - July 4)

3. Update existing references (July 13 - August 2 , 3 weeks)

The following functions use generation number or committer date for cutoffs:
- `contains_tag_algo`
- `can_all_from_reach_with_flag`
- `paint_down_to_common`
- `remove_redundant`

All of these can be updated with little to no changes.

--> Second evaluation (July 27 - July 31)

### Extend generation number to --ancestry-path (August 3 - August 16, 2 weeks)

`--ancestry-path A..B` returns the commits that are both descendants of A and
ancestors of B. The algorithm computes commits that are reachable by B but not
by A, reverse the graph and walk backwards from A [12].

We can use generation number to limit walks to find commits reachable by B.

[12]: https://lore.kernel.org/git/0c6b42e4-e825-ff70-a528-e118abf4c435@gmail.com/

### Examine performance improvements (August 17 - August 24, 1 week)

- Compare the performance of generation number v2 versus topological level using
  performance test suite.
- Wrap up any existing patches.

The performance report would be a nice summary my work over the summer.

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

## Post GSoC

I would love to keep contributing to git after the GSoC period ends. There`s so
much to learn from the community.

Hannes`s comment on checks as a penalty that should be paid only by constant
strbufs was a perspective I had not considered [13].

Interacting with Kyagi made me rethink the justifications _emphasizing commit
messages_. I was at my wit`s end, which makes me appreciate my patient mentors
more and want to give back to the community.

[13]: https://lore.kernel.org/git/467c035f-c7cd-01e1-e64c-2c915610de01@kdbg.org/

## About Me

I am Abhishek Kumar, a second-year CSE student at National Institute of
Technology Karnataka, India. I have a blog where I talk about my interests -
programming, fiction, and literature [14].

I primarily work with C/C++ and Ruby on Rails. I am a member of my institute`s
Open Source Club and student-built University Management System, _IRIS_. I have
some experience of mentoring - Creating their code style guide and being an
active reviewer [15].

[14]: https://abhishekkumar2718.github.io/

[15]: https://iris.nitk.ac.in/about_us

## Availablity

The official GSoC coding period runs from June 1 to August 24.

Due to the outbreak of COVID-19 in my country, my college has pre-emptively
announced summer vacations from March 17 to June 1. Unfortunately,
I would have classes for a large part of the coding period. However, I can still
contribute 35-40 hours every week due to a low course load (~20 hours a week).

I would not be able to contribute from June 29 to July 4 due to end semester
exams.  It would be easily compensated during the subsequent "semester break"
from July 5 to July 27.

## Contact Information

```
     Name: Abhishek Kumar
    Email: abhishekkumar8222@gmail.com
    Major: Computer Science And Engineering
   Degree: Bachelor of Technology
Institute: National Institute of Technology Karnataka
   Github: abhishekkumar2718
 Timezone: UTC+5:30 (IST)
```

Thank you for taking the time to review my proposal!
