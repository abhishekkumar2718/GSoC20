# Convert mergetool to builtin

## Synopsis

A few subcommands of git are in the form of shell and Perl scripts. This causes
problems in production code - in particular, on multiple platforms.

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

> Add justification for mergetool over mergetool--lib

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

## Conversion of difftool

## Dependency Graphs

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

1. Community bonding period (April 27 - May 18)
- Study mergetool in greater detail.
- Read up on builtin, run-command and other git internals.
- Understand the test suite.

2. Create a skeleton builtin (May 18 - May 21)
- Rename git-mergetool.sh to git-legacy-mergetool.sh
- Add a configuration variable mergetool.useBuiltin
- Add a builtin which executes the legacy-mergetool unless mergetool.useBuiltin
  is true

3. Implement scaffolding (May 21 - May 31)
- Convert `main` except assigning mergetool
- Around 100 lines

4. Implement shared functions (June 1 - June 7)
- Convert `mergetool_tmpdir_init`
- Convert `cleanup_temp_files`
- Convert `describe_file`
- Convert `checkout_staged_file`
- Around 80 lines

5. Teach builtin to resolve symlink conflict (June 7 - June 20)
- Convert `merge_file`
- Convert `resolve_symlink_merge`
- Around 150 lines

I noticed a possible bug in `resolve_symlink_merge`. The original file is
backed up regardless of the configuration settings. Is that intended behavior? [7]

[7]: https://github.com/git/git/blob/076cbdcd739aeb33c1be87b73aebae5e43d7bcc5/git-mergetool.sh#L92

--> June 15 - June 19: Phase 1 evaluation

6. Teach builtin to resolve deleted file conflict (June 20 - June 27)
- Convert `resolve_deleted_merge`
- Around 70 lines

7. Teach builtin to resolve submodule conflict (June 27 - July 10)
- Convert `stage_submodule`
- Convert `resolve_submodule_merge`
- Around 125 lines

The implementation of `resolve_submodule_merge` seems repetitive. It might be
possible to streamline both cases using flags and swapping variables [8].

[8]: https://github.com/git/git/blob/076cbdcd739aeb33c1be87b73aebae5e43d7bcc5/git-mergetool.sh#L154

--> July 13 - July 17: Phase 2 evaluation

8. Teach builtin to assign merge tool (July 10 - July 15)
- Convert `get_configured_merge_tool` from mergetool--lib
- Around 50 lines

Since the builtin would execute the helper script for each file conflict,
querying config every time would be inefficient.

--> My college begins from July 20

9. Teach builtin to shell out for file conflict (July 15 - July 31)
- Write a minimal mergetool--helper.sh (similar to difftool--helper.sh)
- Call the helper script from the builtin
- Retire the legacy script.

This helper script would:
- Call `guess_merge_tool` from mergetool--lib.sh if mergetool has not been set
- Call `run_merge_tool`

The builtin would take care of backup and clean-ups.

10. Teach builtin to not trust exit code (August 1 - August 10)
- Convert `trust_exit_code`, `run_merge_cmd`, `check_unchanged` from
  mergetool--lib
- Around 50 lines.

11. Wrap up (August 10 - August 17):
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

_Pack bitmap support for libgit2_ and _Replace object loading/writing layer by
libgit2_, two of 2014 project ideas were also interesting - although I didn't
look into their current status.

## Potential Problems

### Introduction of new bugs

Rewriting code always has the possibility of introducing new bugs. The test
suite groups together all types of conflicts together. Therefore the test suite
would be ineffective until all types are implemented.

While straightforward bugs would be caught by the test suite, the project might
end up with subtle bugs and unspecified behavior.

The choice of using mergetool.useBuiltin comes in handy here. We could release
early preview versions and fix any bug reports we get. Once confident in the
builtin, we could ship it by default.

### Performance

_Make the common case fast_.

When it comes to mergetool, the typical case is overwhelmingly file conflicts.
Until mergetool--lib is converted, builtin would execute helper script which
would in turn source mergetool--lib and call `run_merge_tool`.

This is more work than the script version - which would just source
mergetool--lib  and call `run_merge_tool`.

On the other hand, the other three cases are undoubtedly faster - since they
would be entirely in C.

It would be hard to predict whether performance, as perceived by users, would
improve and by how much. Builtin would likely perform better for a constant
number (_K_) of file conflicts and worse when there are more than K file
conflicts.

However, the inclusion of difftool gives me hope - since both difftool and
mergetool suffer from identical penalties.

I _might_ be worrying over microseconds of performance here ;).

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
programming, fiction, and literature [9].

I primarily work with C/C++ and Ruby on Rails. I am a member of my institute's
Open Source Club and student-built University Management System, _IRIS_. I have
some experience of mentoring - Creating their code style guide and being an
active reviewer [10].

[9]: https://abhishekkumar2718.github.io/

[10]: https://iris.nitk.ac.in/about_us

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
strbufs was a perspective I had not considered [11].

Interacting with Kyagi made me rethink the justifications _emphasizing commit
messages_. I was at my wit's end, which makes me appreciate my patient mentors
more and want to give back to the community.

[11]: https://lore.kernel.org/git/467c035f-c7cd-01e1-e64c-2c915610de01@kdbg.org/

## Contact Information

| Name      | Abhishek Kumar                             |
| Major     | Computer Science And Engineering           |
| Institute | National Institute Of Technology Karnataka |
| E-mail    | abhishekkumar8222@gmail.com                |
| Github    | abhishekkumar2718                          |
| Timezone  | UTC+5:30 (IST)                             |

Thank you for taking the time to review my proposal!