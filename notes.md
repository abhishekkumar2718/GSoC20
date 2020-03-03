## Project ideas

> I am going through project ideas from earlier GSoC and outreachy applications to see if I like something.

- Pack bitmap support for libgit2.
- Teach git stash to handle unmerged index entries.
- Port a script to builtin.

Improve consistency of sequencer commands:
- Final report: https://rashiwal.me/2019/final-report/
- Proposal: https://rashiwal.me/git_proposal_19.pdf

Make pack access thread safe:
- Proposal:
- Report:

## Convert interactive rebase to C:
- Proposal: https://github.com/agrn/gsoc2018/blob/master/proposal/git-proposal-final.pdf
- Repository: https://github.com/agrn/gsoc2018
- Blog: https://blog.pa1ch.fr/category/gsoc-2018.html

His repository is a gold mine and I have been digging through code. Do refer to his proposal while drafting my own.

Patch 1:

rebase--interactive.sh has large overlaps with code specialized with preserve merges, splitting which makes the conversion process simpler.
`preserve-merges` is set to be deprecated, so converting it does not make much sense.

- Add git-rebase--preserve-merges to gitignore
- Add git-rebase--preserve-merges to Makefile
- Move relevant code from git-rebase--interactive.sh to git-rebase--preserve-merges.sh
- Clean out git rebase.sh

Functionally, there are no changes. Code is moved from a shell script to another.

Patch 2:

Rewrite append_todo_help().

- Create new files `rebase-interactive.c` and `rebase-interactive.h` to store specialized functions.
- Add `rebase-interactive.o` to Makefile.
!!!
- Remove append_todo_help() call from shell and call `rebase--helper` with appropriate options.
!!!

This is the magic trick of conversion process.

1. Create a helper C file and implement functions piecewise.
2. Replace calls within the shell script to the helper -> This ensures that git builds at each step.
3. Remove the script and rename helper once all calls are implemented.
4. Modify build process so that everything works.

Patch 3:

Rewrite edit_todo().

- Modify some settings related to editor.
- Implement edit_todo() in rebase-interactive.c.
- Add a flag to trigger edit_todo in `rebase--helper.c`
- Replace call in shell script with `exec git rebase--helper --edit-todo`

* Use getpid() while debugging so that the process flow works as expected.

- Maintain a long running integration branch instead of small patches. Avoids the hell of rebasing and sending emails every so often.

## Convert interactive rebase to C
- Proposal: https://github.com/prertik/GSoC2018/blob/master/GSoC2018proposal.pdf
- Repository: https://github.com/prertik/GSoC2018/
- Despite my expectations - Prertik's code is haphazard and definitely not an ideal to follow. I like alubin's approach much more.

## Convert git stash to builtin
- Repository: https://github.com/ungps/gsoc2018

## Incremental rewrite of git-submodules
- Drive: https://drive.google.com/drive/folders/0B1HuYKpJFTqWYk5vcmpaa2JJa28
- Report: https://docs.google.com/document/d/1RmUvJBf4x8TI71Fltg8xWP-s7zkhz3bGPyEJMgRx91Y/edit

## Incremental rewrite of git bisect
- Drive: https://drive.google.com/drive/folders/0B2SBIDYEkAaiV2xhNWZNSUdWOVE
- Report: https://docs.google.com/document/d/1Uir0a8cRYlWANuzoU4iTDtEvPukvtTJcC_dB3KJUgqM/edit

> Read up on the process of creating builtin - builtin.h

> mergetools should not be a part of main binary and must be implemented as `mergetools.c`.

> Read up on my first contribution which covers the process of implementing a function.

> Go through the test suite

> Include links to the mailing-list discussions related to project chosen.

> Include links to previous drafts of application

> Johannes earlier work on conversion of difftool is important!

> Johannes, Hannes, David, Denton are possible co-mentors/reviewers.

> Create an example repository as in test suite to see how it works.

---

Shell timings: {real: 12.946s, user: 9.576s, sys: 4.229s}
C timings:
