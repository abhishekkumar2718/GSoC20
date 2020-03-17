## Commit Graph

A commit contains:
- Hash of root tree
- List of parent commit hashes: No parent -> root commit, two or more parents ->
merge commit. Parent order matters.
- {Name, Email, Time} of {Author, Committer}. Times are stored as unix time.
- Commit message.

> Add notes from git-commit-graph file.

If there are no parents, generation number for commit is 1. Otherwise, it is one
more than the maximun of generation numbers of its parents.

If gen(A) < gen(B), then A cannot reach B.

Both articles are cool but not really relevent. I am going to read up on the
papers now.

## Changes in commit-graph format

Well, we have a 1-byte version number following the "CGPH" header in
the commit-graph file, and clients will ignore the commit-graph file
if that number is not "1". My hope for backwards-compatibility was
to avoid incrementing this value and instead use the unused 8th byte.

However, it appears that we are destined to increment that version
number, anyway. Here is my list for what needs to be in the next
version of the commit-graph file format:

1. A four-byte hash version.

2. File incrementality (split commit-graph).

3. Reachability Index versioning

4. Move from SHA-1 to SHA-256

Most of these changes will happen in the file header. The chunks
themselves don't need to change, but some chunks may be added that
only make sense in v2 commit-graphs.

## TODO:
- Read linked resources.
- Read git internals related to commit, commit-graph.
- Look up other reachablity algorithms.
- The google collab link is amazing. It implements most of algorithms in python
I should definitely look up their implementation when writing pseudo code.

----

Derrick's discussion: https://public-inbox.org/git/86tvl0zhos.fsf@gmail.com/t/
Generation number: https://devblogs.microsoft.com/devops/supercharging-the-git-commit-graph-iii-generations/
Preach: https://arxiv.org/abs/1404.4465
Ferrari: https://github.com/steps/Ferrari, https://arxiv.org/abs/1211.3375
Jakub's implementation: https://colab.research.google.com/drive/1V-U7_slu5Z3s5iEEMFKhLXtaxSu5xyzg
Split commit graph: https://devblogs.microsoft.com/devops/updates-to-the-git-commit-graph-feature/
Bitmap optimization: https://githubengineering.com/counting-objects/
