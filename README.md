# guiltix - (potential) prototype tool for modern patchqueue handling

## Base principles

* patchqueue in patch files is used as a way of exchanging the state
  of the queue, and following the changes of the patchqueue over time
* git commits are used as the primary way to work on the patch queue,
  and are sync'd back and forth at user request from/to the patch
  files

### Why a patchqueue?

"Why would a lowly patchqueue be a better choice than a nice Git
branch?  Or a [reintegration
branch](https://github.com/felipec/git-reintegrate/)?" is a question
that comes quite often, and I was myself a proponent of using Git
branches... until I was pointed out to a number of limitations.  Let's
try to list the most compelling things we want to have, and that can't
easily be done with simple branch:

* maintaining meta-information about the patches: why we use them,
  what's their status in the queue (backport, upstreamable (with xref
  to the PR), non-upstreamable, custom stuff with the idea of
  upstreaming when it's cleaned up enough, etc).  They can also be
  easily commented out when needed (and with due description) and
  commented out again to bring them back in a snap.

* rebasing a part of the work and handing it over to a dev knowing
  better the area of code touched by the next patches: doing that with
  branches requires to setup specific conventions, and the result
  lacks the proper audit trail.

* documenting how the patch series evolves over time: a natural fit is
  the description for a commit that brings the queue from one revision
  to another.  StGit is able to link successive branches that way
  without going through patch files, but the tooling around this is
  heavy (or at least was, when I used it intensively around 2006).
  OTOH a (properly-crafted) git history of patchfile does not require
  any special tools to browse, other than `git diff` and friends.


## Comparisons with prior art

### guilt

Structural pros:
* patches are actually the first-class citizens, which allows for
  collaboration and some understanding of history
* handles any patch, not just a `git-format-patch` output (or is that
  actually a con?)

Structural cons:
* overloaded interface (32 subcommands)
* not single purpose (`guilt commit` making a commit "official" and
  out of stack), but different nature of commands not obvious
* commits for unapplied patches are not tracked
  * => patches must be reapplied on pop/push
* conflict resolution left as exercise to the user
  * `.rej` files from good old `patch`, 1970-style
  * still creating a commit with the resolved part, untracked `.rej`
    files as only reminder something did not go as expected, possible
    to create/new patches etc
* does not track which commit patches are supposed to apply to
* unapplied patch names not available in git, must look them up in
  `series` for a `guilt delete`
* comments in `series` file not in git: must be manually removed there
  after a patch section they comment gets fully deleted
* bad interactions with trivial git workflows (git commit)
* ...?

Contingent cons:
* antiquated UI with extremely poor discoverability (`guilt new
  --help` creates a new patch with that name)
* does not easily allow to share a single clone for different
  patchqueues in the same repo (cannot switch branch easily because of
  single status file)

### stgit (WIP)

Refs:
* https://wiki.xenproject.org/wiki/Managing_Xen_Patches_with_StGit

Structural pros:
Structural cons:
Contingent cons:

### gbp-pq, gbp-pq-rpm (WIP)

Refs:
* https://honk.sigxcpu.org/projects/git-buildpackage/manual-html/gbp.patches.html

Structural pros:
* limited interface (6 subcommands, only 3 essential: import, export,
  apply), and let use plain git for everything not directly related to
  the patchqueue work

Structural cons:
* restricted to a single developer doing one full rebase of the
  patchqueue, no way to share a partial rebase
* not general-purpose, needs minimal structure of a Debian source
  packages (patchqueue in `debian/patches/`, `debian/control` required
  but can be empty; requires `gbp pq import --ignore-new` since
  creating them makes the tree dirty)

Contingent cons:
* automatically drops relevant patches on `gbp pq rebase`, but does not
  seem to drop them from series on `export`?

### gbs (WIP)

https://docs.tizen.org/platform/reference/gbs/gbs-maintenance-models/

https://github.com/intel/gbs
https://docs.tizen.org/platform/reference/gbs/gbs-build/
https://youtu.be/DLaD0hHY_F8?t=109

Structural pros:

Structural cons:
* devs primarily stack commits on dev branch, of which patchqueue is a
  product: focus is on collaboration through a git branch, at the
  expense of patchqueue cleanness.

Contingent cons:

### git-series (WIP)

* https://www.youtube.com/watch?v=xJ0DBaHnlQ8
* https://wiki.xenproject.org/wiki/Managing_Xen_Patches_with_Git-series

## Needed workflows

### single user

What: user starts working on a patchqueue stored in a separate git
repo, imports original patchqueue on the original base commit

How:
```
$TOOL import $PATCHDIR
git rebase -i
$TOOL export
git -C $PATCHDIR push
```

* `$TOOL import -b $QUEUEBRANCH`:
  * expects a clean git-controlled workspace in $PATCHDIR
  * creates $QUEUEBRANCH on base commit advertised in $PATCHDIR available in $PWD
* `$TOOL export` takes care of `git add`, `commit`

### stashing a rebase

What: user takes a break from the rebase, e.g. to switch to another
branch or have a weekend

How:
* `$TOOL export` saves
  * rebase instruction sheet
  * all local commits, included previous versions of already-rebased
    patches necessary to reproduce all not-yet-rebased ones
  * expects *all* base commits of successive patches to be available
* `$TOOL import` reconstructs the git rebasing state

### sequential multi-user rebase

What: knowledge of patched areas is spread among several persons, who
work one after the other on their part of the patchqueue

How: In turn, each user pulls $PATCHDIR, and pushes when finished.

### reconciliating rebase with concurrent changes

What: changes were pushed to master while (some part of) the rebase
was done, user needs to rebase the pathqueue changes onto new master

How:
* patchqueue must be exported first, so we have 2 proper $PATCHDIR
  branches, and ours can be rebased on new master
* patches already applied in the pq branch must be rewound
