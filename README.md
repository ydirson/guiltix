# guiltix - (potential) prototype tool for modern patch queue handling

## Context

A distro makes packages from source code coming from upstream
projects.  But it often needs to apply patches to that upstream
source, be it for adapting to the unique context of the distro, fixing
bugs, including new features that, even when not ready for
upstreaming, are required in the distro's context, etc.

This is typically done by including patch files alongside the upstream
source (though sometimes an already-patched source tree is used).

### Interlude: a word on already-patched source trees

Assuming the patched source is under revision control, there are
basically 2 approaches in this field: keeping a single "downstream"
branch in which patches are committed and into which new upstream
versions are merged (that is, a fork), and regularly rebasing a branch
onto newer upstream versions.

First it should be noted that both approaches share a common downside:
to get a single fix into the package, the natural workflow implies 3
separate publishing steps: commiting and pushing in the upstream fork,
creating a new archive and publishing it so the RPM build can access
it, and updating the RPM packaging.  This can be brought down to 2
separate publishing steps if tooling is in place so the RPM packaging
can directly pull from the fork (TBW: need examples of who does that).

The fork approach does not allow (AFAIK) for nice upstreaming
workflows, since the difference (at Git commit level) from upstream to
the fork will *always* be seen as including all the commits ever
included in the fork, regardless of whether a version thereof was
upstreamed.  Exemples of this approach are numerous in the embedded
space, with forks of the Linux kernel where `git describe` announces
tens of thousands of commits added to an official Linus tag.

The "patch queue rebase" approach has the additional downside that
reviewing the source-code differences has to be done in a different
repository, and more aspects are discussed in more details later in
this document, in "Why not just use a branch?".

That said, one case for which already-patched source trees have a
clear edge today over a "patch queue as patchfiles", is when the
upstream source includes binary files that need to be modified.
Although packaging systems usually can handle this through additional
source files, the tooling to deal with it is basically not here today.

### The patch queue

What we call "patch queue" is thus a collection of single patches and
of patch series, each with their own topic.  This does not presume
anything on its representation (even though it is often conflated with
the "a patch queue as patchfiles and a `series` file" format
popularized by `quilt` - we'll use "quilt patchqueue" for this
particular one).

As the packaging of a given upstream version gets revised, new patches
and series get inserted into the queue, some get replaced, some get
dropped.  And when the packaging gets updated to a newer upstream
version, a lot of changes can be needed.

And those changes need to be easily auditable, a-priori by reviewers
during the packaging update process, or a-posteriori by investigators
trying to understand the reason why the patched code is in its current
state.


## Base principles

* provide a simple audit trail for the evolution of a patch queue

* patchqueue in patchfiles-plus-an-index is used as a way of
  exchanging the state of the queue, and following the changes of the
  patchqueue over time.  There may be needs for several metadata
  formats, both for patchfiles and index, for interoperation with
  other tools (`git format-patch`, quilt patchqueue (essentially
  including DEB patchqueue), SRPM-compatible `Patch:` statements,
  BitBake `SRC_URI` patch list, ...).  This allows to use the
  packaging data as single source of truth (much like how `gbp pq` and
  `gbp pq-rpm` work)

* git commits are used as the primary way to work on the patch queue,
  and are sync'd back and forth at user request from/to the patch
  files

### Understanding the domain

Let's step back for a moment: information to understand the code (a
`tree` in Git, which would be "abstraction level 0") belong to the
code files themselves; information to understand the changes
(aka. level 1, a `commit` in Git) that we make to the code tree belong
to the commit message: we don't want the latter in the code, just as
we don't want information to understand the code available solely in
the commit message.

Then on top of these two concepts, lies the pachqueue itself (making
it abstraction level 2). It is more than the mere sequence of commits:
in many cases it is a curated list of commits, some
cherry-picked/packported from upstream (grouped by the release that
sports them to prepare future rebases), maybe several patch series for
distinct features, submitted upstream or not yet ready for upstream,
even custom ones not meant to be upstreamable.  Then there are many
informations we may want to keep there (why we took them, how they
articulate with other parts of the patch queue, ...).  One prominent
example of a metadata-heavy patch queue is [the Xen patchqueue from
XenServer](https://github.com/xenserver/xen.pg/blob/master/patches/series).

How this set of metadata changes over time is crucial, just as much as
it is for the code in the upstream project.  This brings us another
level of abstraction, the patch queue history, which we'll label
"level 3" - and hopefully be the last one we'll need :)

Now what tools do we have to deal with this complexity and make it
bearable?

* code: editors and IDEs, essentially solved
* code history: version control software, essentially solved
* patch queues:
  - file-based formats involving patchfiles
  - various tools and workflows to use a VCS (StGit, gbs, others?)
* patch queue history:
  - standard VCS history of file-based formats
  - use of git commits with special ref namespace, to record ancestry
    of a patchqueue-as-git-branch (StGit 0.1x - should check if still
    the case)
  - any other?


### Why a patchfiles-and-index and not just use a branch?

"Why would a text patchqueue be a better choice than a Git branch?  Or
a [reintegration
branch](https://github.com/felipec/git-reintegrate/)?" is a question
that comes quite often, and it's quite natural, as Git is a more
modern tool than diff/patch.  I was myself a proponent of using Git
branches... until I was pointed out to a number of limitations.

But first, one thing to keep in mind is that we're talking about the
"canonical storage" or "exchange format" of the patch queue.  There's
no question that Git is a vastly useful tool for *reworking* the patch
queue, much like we all agree that a tree of text files is a suitable
"canonical storage" or "exchange format" for code, and people can use
powerful IDEs to manipulate them.

We *can* find a way to encode this patch queue history (level 3)
information inside code history and in the patchqueue-as-branch
(levels 1 and 2) (e.g. include it inside commit messages, or use empty
commits or merge commits to add meta-information).  But that's similar
to describing inside the code the changes that were done since
previous versions of that code: indeed we used to do that sometimes
when VCSs were non-existent or too heavy to use them like we do now,
and had no practical place to store that information.

Let's try to list the practical differences between the patchqueue
approaches above.  In fact the alternatives for patch queues and their
histories are tightly tied together: you cannot use standard VCS tools
on it, unless the data you manipulate is in files, and it's very
uncomfortable when those are not text files (read: at best, needs
extra tooling).  And similarly the StGit approach only makes sense
when a Git branch is the primary storage for the patch queue.

So let's reformulate the options we want to compare:

- file-based formats involving patchfiles, stored in a VCS tree
- StGit model
- gbs model?

Having a textual description of the sequence of patches that make a
patch queue makes things easier for several use cases:

* maintaining meta-information about the patches: why we use them,
  what's their status in the queue (backport, upstreamable, etc).
  When encoded in a text file they can also be easily modified,
  commented out when needed (and with due description) and commented
  out again to bring them back in a snap, all with standard Git tools.
  
  Encoding them in Git commits requires more overhead to modify them,
  and special tooling to get the history of their changes.

* documenting how the patch series evolves over time: a natural fit is
  the description for a commit that brings the queue from one revision
  to another.  StGit is able to link successive branches that way
  without going through patch files, but the tooling around this is
  heavy (or at least was, when I used it intensively around 2006).
  OTOH a (properly-crafted) git history of patchfile does not require
  any special tools to browse, other than `git diff` and friends.


## Comparisons with prior art

### guilt

Tool summary:

* maintains a patch queue as patchfiles and a `series` plain-text
  index (which may include comments)
* to work on them, applies them on top of original code as git commits
* allows to pop/push patches to go edit them, not designed to let user
  use other git tools on the same branch, rather provides its own
  tools instead
* does not cover handling the history of the patch series, this is
  trivial to do with Git (so user has full control of what gets
  tracked) (and this covers merging different history lines, as text)

Structural pros:
* patches are actually the first-class citizens, which allows for
  interoperability, collaboration and some understanding of history
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

Tool summary:

* maintains a patch queue both as git commits, and as textual
  patchfiles with a `stack.json` index
* automatically records every single patchqueue change under a
  separate git ref under `stacks/`, including push/pop (ie. some sort
  of stack reflog, not meant for sharing)

Refs:
* https://wiki.xenproject.org/wiki/Managing_Xen_Patches_with_StGit

Structural pros:
Structural cons:
Contingent cons:

### gbp-pq, gbp-pq-rpm

Tool summary:

* maintains a patch queue as patchfiles and the DEB/RPM patch index
  expected by the source-package format
* lets user import into a git branch and export back to package
  format, and use plain git for everything not directly related to the
  patchqueue work

Refs:
* https://honk.sigxcpu.org/projects/git-buildpackage/manual-html/gbp.patches.html

Structural pros:
* limited interface (6 subcommands, only 3 essential: import, export,
  apply)

Structural cons:
* restricted to a single developer doing one full rebase of the
  patchqueue, no way to share a partial rebase
* not general-purpose:
  - `gbp-pq` is tied to the Debian source packages structure
    (patchqueue in `debian/patches/`, `debian/control` required but
    can be empty; requires `gbp pq import --ignore-new` since creating
    them makes the tree dirty)
  - both assume the packaging git branch and the upstream git branch
    live in the same repository, making it not possible (or at least
    very impractical) to have several packages in a single "monorepo".
    This is reasonable given the structure of a DEB source tree (which
    has to include the unpacked upstream source), but makes the
    approach not suitable for a Yocto-like packaging repo, which can
    be applied to RPM.
* (or is it contigent?) `gbp-pq-rpm` does not preserve import patch
  names on export, and relocates all Patch clauses in specfile so they
  are grouped.  It does not loose the previously-interpersed comments
  but looses any link with the patches they describe.

Contingent cons:
* automatically drops relevant patches on `gbp pq rebase`, but does not
  seem to drop them from series on `export`?
* `gbp-pq-rpm` (indeed `gbp.patch_series`) requires specfile and
  patches to be in the same dir: no support (in 0.9.38) for
  formely-common `SPECS/` + `SOURCES/` layout

Other notes:
* upstream seems to be stalled (and their cgit server dying)
* `gbp-pq-rpm` support for `%if/%ifarch` assumes no interaction
  between conditional patches and the rest (developement work is
  always done with all patches applied, the conditional gets into the
  commit message).  This reserves the `%if` feature for very specific
  usage when the patch *should absolutely not* be applied in the other
  cases - which is not unreasonable.

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

Tool summary:

* main focus is support is supporting patch-series history, for
  preparing successive versions of a patch series to be submitted to
  an upstream project
* meant for a single topic, so lacks support for pq metadata?
* implemented as extra tooling to store the series' history in git
  refs.  Lets user manipulate the series tree (add to index, commit a
  new revision of the series) when it fits
* development stalled, last commit in 2024, last release published in
  2019

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
