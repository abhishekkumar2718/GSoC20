# Convert mergetool to builtin

> This proposal has been autoconverted to PDF. Refer to original mailing list
> discussion [here](https://lore.kernel.org/git/20200325170023.GA115466@Abhishek-Arch/).
> You can also read this proposal on [Github](https://github.com/abhishekkumar2718/GSoC20/blob/master/mergetool.md).

## Synopsis

A few subcommands of git are in the form of shell and Perl scripts. This causes
problems in production code - in particular, on multiple platforms among others.

This project rewrites git mergetool in C to improve its performance and
portability. It would also lay the groundwork for the subsequent conversion of
mergetool--lib.

## About git mergetool

Git mergetool runs merge conflict resolution tools to resolve merge conflicts.
Internally, it is implemented as two scripts - `git-mergetool.sh` and
`git-mergetool--lib.sh`.

`git-mergetool.sh` is the driver script and does the following:
- Parse options.
- Get merge tool name from configuration.
- List unmerged files.
- Identify the type of conflict.
- Resolve deleted, submodule, and symlink conflicts.
- Pass normal file conflicts to `git-mergetool--lib.sh`.

`git-mergetool--lib.sh` stores common functions shared by `git-mergetool.sh`
and `git-difftool--helper.sh` and does the following:
- List all conflict resolution tools.
- Set up tools.
- Validate conflict resolution in case of untrustable tools.
- Run the merge/diff tool.

## Goal

At around 1500 lines of code, conversion of the mergetool-related scripts is
impossible over a summer of code project.
[git-mergetool.sh: 529, git-mergetool--lib.sh: 463, mergetools/: 472]

The goal of this project is to rewrite `git-mergetool.sh` in C. Normal merge
conflicts would still be resolved through `git-mergetool--lib.sh` (a strategy
adopted by difftool as well). I hope future SoC/Outreachy students pick up on
this idea and rewrite the other two scripts.

## Benefits to the community

### Better performance

Subcommands written in shell scripts are slower than builtins. Shell scripts are
inherently slower than binaries and shell scripts invoke git's porcelain
commands, which do not have access to git's internal API. For each such call,
git would re-read configuration files, repository index, etc. Such repetition
is inefficient.

As noted in Hannes's patch, git-mergetool _spawns an enormous number of
processes_ [1]. The test suite spawns over 12,000 processes and 2,000 non-git
commands.

Partial conversion for difftool improved performance by 4.3x for Linux and 1.2x
for windows [2]. We can expect similar gains for mergetool as well.

Improvements differ due to the overhead from shelling out to helper script.
A complete conversion would avoid the overhead and show even more significant
improvements for both systems.

[1]: https://lore.kernel.org/git/cover.1560152205.git.j6t@kdbg.org/
[2]: https://lore.kernel.org/git/8ab75685f718cfeb571409830ae3c6ee14ac5158.1484857756.git.johannes.schindelin@gmx.de/

### Portablity

Shell scripts often rely on POSIX utilities. They are not necessarily available
natively on all platforms or might have some differences. On non-POSIX platforms
(like Windows), utilities need to be included along with an emulation layer. C
offers improved portability.

### Conversion of mergetool--lib

As mentioned earlier, conversion of the mergetool-related scripts has to be
spread over 2-3 SoC or similar projects due to the size of scripts involved.
Conversion of mergetool would set up most of the plumbing required for
mergetool--lib and makes the subsequent conversion possible.

As Christian pointed out, it might make more sense to convert mergetool--lib
before converting mergetool. However, mergetool takes a greater priority (in my
opinion) because mergetool makes many more calls to git subcommands than
mergetool--lib.  The performance would improve from both moving from bash to C
and use of internals.

On a broader (_and possibly ambitious_) note, I would be happy to co-mentor
any student who takes up the conversion process. It would be gratifying to see
our collective efforts finish a mammoth task.

## Related Work

Back in 2016, Johannes worked on a remarkably similar "project" - converting
`git-difftool.sh` into a builtin [3]. The conversion is described below.

There have been similar SoC/Outreachy projects converting other scripts:
- bisect--helper by and Miriam Rubio.
- Interactive rebase by Alban Gruin and Pratik Karki [4], [5].

and others.

[3]: https://lore.kernel.org/git/cover.1479834051.git.johannes.schindelin@gmx.de/
[4]: https://github.com/prertik/GSoC2018/
[5]: https://github.com/agrn/gsoc2016

## Overview

_This section is an oversimplified primer on how mergetool works internally._

git-mergetool runs conflict resolution tools to resolve merge conflicts.

In a merge conflict, the following files are involved:
- Local: The 'ours' side of the conflict i.e., current HEAD.
- Remote: The 'theirs' side of the conflict i.e., branch merging into HEAD.
- Base: The common ancestor of both branches.

Merge conflicts are of four types - Symbolic link conflict, deleted file
conflict, submodule conflict, and file conflict.

First three types of conflict occurs when either local or remote is a symlink,
deleted file or part of a submodule.

Checking out the appropriate version of the file from the index resolves
symlink conflicts.

Deleted file conflicts are resolved by either adding file back to index or
removing it from the working tree and index.

Submodule conflicts are somewhat involved. Assuming the user wants to keep
local file:

```
if the local file exists in current directory
  - If local is in submodule mode, stage the submodule.
  - If remote is in submodule mode, check out the file from the index with
stage 2.
else:
  - If local file exists in a subdirectory, add it to index.
  - If local file does not exist, force remove it from the index.
```

There is a similar flow if the user wants to keep the remote file.

File conflicts arise when competing changes are made to the same line of a file.
Git merge's strategies cannot solve them, and they must be resolved manually.

Mergetool relies on external tools like vimdiff, kdiff, meld to resolve
conflicts. Mergetool decides the external tool in the following precedence:
- Parameter passed
- Configuration
- Iterating over defaults

`get_merge_tool` (defined in mergetool--lib) decides the external tool. This
function is used by both mergetool and difftool--helper.

The return code of the external tool is usually not trusted. Depending on
whether we trust return code or not, the script prompts the user to re-affirm
whether the merge was successful.

The main function of mergetool iterates over the unmerged files (in given order
if passed) - identifying the conflict type and calls the appropriate function
to resolve.

### Conversion of difftool

The conversion had the following three patches:

1. difftool: add a skeleton for the upcoming builtin
- Rename git-difftool.perl to git-legacy-difftool.perl
- Create builtin/difftool.c which executes git-legacy-difftool.perl
- Add difftool to builtin.h
- Add "difftool.useBuiltin" configuration option
- Modify build process

2. difftool: implement the functionality in the builtin

3. difftool: retire the legacy script
- Remove git-legacy-difftool.perl from build process
- Remove outdated "NEEDWORK" comments
- Remove perl dependency from test file

Since most of the conversion was done in a single commit, it's hard to talk
about the finer order of changes.

Similar to this, my changes will be grouped as:

1. Create a skeleton builtin.
2. Core functionality: Implement scaffolding, implement shared functions, teach
builtin to resolve symlink, submodule and deleted file conflicts, and others.
3. Teach builtin to shell out for file conflict (at which we retire
mergetool.sh)

## Plan

Similar to the conversion of difftool, I plan to create a builtin that shells
out to a helper script. Once mergetool--lib is converted, we can retire the
helper script and conversion would be complete.

I realize this is unlike most conversions, where the script calls the builtin,
and features are incrementally transferred.

My choice is motivated by the fact that the child process cannot set variables
for their parent. mergetool makes extensible use of setting variables to share
them between functions.

For example - If I implement `git mergetool--helper --tmpdir-init` to replace
`mergetool_tmpdir_init` [6], I cannot set `$MERGETOOL_TMPDIR`. One possible
workaround (which does not account for "returning" multiple variables) is to
print out results and capture it in the script. But it seems too hacky to me.

[6]: https://github.com/git/git/blob/076cbdcd739aeb33c1be87b73aebae5e43d7bcc5/git-mergetool.sh#L41

I plan to break down the implementation into following smaller steps:

1. Community bonding period (May 4 - June 1)
- Study mergetool in greater detail.
- Read up on builtin, run-command and other git internals.
- Understand the test suite.

2. Create a skeleton builtin (June 1 - June 4, 3 days)
- Rename git-mergetool.sh to git-legacy-mergetool.sh
- Add a configuration variable mergetool.useBuiltin
- Add a builtin which executes the legacy-mergetool unless mergetool.useBuiltin
  is true

3. Implement scaffolding (June 4 - June 10, 10 days)
- Convert `main` except assigning mergetool
- Around 100 lines

4. Implement shared functions (June 10 - June 17, 7 days)
- Convert `mergetool_tmpdir_init`
- Convert `cleanup_temp_files`
- Convert `describe_file`
- Convert `checkout_staged_file`
- Around 80 lines

5. Teach builtin to resolve symlink conflict (June 17 - July 1, 14 days)
- Convert `merge_file`
- Convert `resolve_symlink_merge`
- Around 150 lines

--> Phase 1 evaluation (June 29 - July 3)

--> End semester exams (June 29 - July 4)

6. Teach builtin to resolve deleted file conflict (July 1 - July 8, 7 days)
- Convert `resolve_deleted_merge`
- Around 70 lines

7. Teach builtin to resolve submodule conflict (July 8 - July 22, 14 days)
- Convert `stage_submodule`
- Convert `resolve_submodule_merge`
- Around 125 lines

8. Teach builtin to assign merge tool (July 22 - July 27, 5 days)
- Convert `get_configured_merge_tool` from mergetool--lib
- Around 50 lines

Since the builtin would execute the helper script for each file conflict,
querying config every time would be inefficient.

Note: Functions like get_configured_merget_tool, guess_merge_tool are only
used by mergetool and can be moved to mergetool builtin.

9. Teach builtin to shell out for file conflict (July 27 - Aug 10, 14 days)
- Write a minimal mergetool--helper.sh (similar to difftool--helper.sh)
- Call the helper script from the builtin
- Retire the legacy script (git-mergetool.sh was renamed to
git-mergetool--lib.sh in an earlier step).

This helper script would:
- Call `guess_merge_tool` from mergetool--lib.sh if mergetool has not been set
- Call `run_merge_tool`

The builtin would take care of backup and clean-ups.

--> Phase 2 evaluation (July 27 - July 31)

10. Teach builtin to not trust exit code (August 10 - August 17, 7 days)
- Convert `trust_exit_code`, `run_merge_cmd`, `check_unchanged` from
  mergetool--lib
- Around 50 lines.

11. Wrap up (August 17 - August 24, 7 days):
- Submit final patches.
- Compare the performance of script and builtin.
- Write a blog summary of the experience.

I have slowed down the speed of conversion in the latter half of the project to
act as a buffer in case of unexpected problems. I might need a week or two to
ensure all tests pass after teaching builtin to shell out for file conflict.

If everything goes well, I could work on converting mergetool specific functions
from `mergetool--lib.sh` - `get_merge_tool_cmd`, `list_merge_tool_candidates`
and others.

I plan to send out patches in the same order. I find that maintaining a
long-running integration branch is more manageable than smaller patchsets.
Smaller, multiple patchsets would suffer from constant rebase and push.

After a "reasonable" break, I am going to look into the conversion of
difftool--helper.

## Potential Problems

### Introduction of new bugs

Rewriting code always has the possibility of introducing new bugs. While
straightforward bugs would be caught by the test suite, the project might end
up with subtle bugs and unspecified behavior.

The choice of using mergetool.useBuiltin comes in handy here. We could release
early preview versions and fix any bug reports we get. Once confident in the
builtin, we could ship it by default.

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
programming, fiction, and literature [7].

I primarily work with C/C++ and Ruby on Rails. I am a member of my institute's
Open Source Club and student-built University Management System, _IRIS_. I have
some experience of mentoring - Creating their code style guide and being an
active reviewer [8].

[7]: https://abhishekkumar2718.github.io/
[8]: https://iris.nitk.ac.in/about_us

## Availablity

The official GSoC coding period runs from June 1 to August 24.

Due to outbreak of COVID-19 in my country, my college has pre-emptively
announced summer vacations from March 17 to June 1. Unfortunately, I would have
classes for a large part of the coding period. However, I can still contribute
35-40 hours every week due to a low course load (~20 hours a week).

I would not be able to contribute from June 29 to July 4 due to end semester
exams. It would be easily compensated during the subsequent "semester break"
from July 5 to July 27.

## Post GSoC

I would love to keep contributing to git after the GSoC period ends. There's so
much to learn from the community.

Hannes's comment on checks as a penalty that should be paid only by constant
strbufs was a perspective I had not considered [9].

Interacting with Kyagi made me rethink the justifications _emphasizing commit
messages_. I was at my wit's end, which makes me appreciate my patient mentors
more and want to give back to the community.

[9]: https://lore.kernel.org/git/467c035f-c7cd-01e1-e64c-2c915610de01@kdbg.org/

## Contact Information

| Name      | Abhishek Kumar                             |
| Major     | Computer Science And Engineering           |
| Institute | National Institute Of Technology Karnataka |
| E-mail    | abhishekkumar8222@gmail.com                |
| Github    | abhishekkumar2718                          |
| Timezone  | UTC+5:30 (IST)                             |

Thank you for taking the time to review my proposal!
