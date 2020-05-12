=======
git-abc
=======
-----------------------------------
Find and manage backport candidates
-----------------------------------

Overview
========

A Backporter's Companion (ABC) aids developers with finding commits in a
fast paced project (upstream) that should be backported to a slower paced
fork (downstream).  git-abc's goal is to provide developers with an easy
and efficient workflow by wrapping git-log(1) up with some additional
filtering and logic into simple commands and by maintaining the selection
state (accept/reject) of backport candidates.  All state is stored in the
git repository using git-notes(1).

Getting Started
===============

Install by simply copying git-abc to somewhere in ``$PATH`` and ensuring
it has execute permissions::

  $ curl https://raw.githubusercontent.com/rhdrjones/git-abc/master/git-abc > ~/Downloads/git-abc
  $ curl https://raw.githubusercontent.com/rhdrjones/git-abc/master/man/man1/git-abc.1 | gzip > ~/Downloads/git-abc.1.gz
  $ install -D -t ~/bin ~/Downloads/git-abc
  $ install -m 444 -D -t ~/.local/share/man/man1 ~/Downloads/git-abc.1.gz

Then, to get started, take a look at the man page ('git help abc') and
follow the steps in the "Workflow" section below.

Workflow
========

1.  Prepare a git repo and set the upstream and downstream pointers.  The
    upstream and downstream pointers may be any two revisions (see
    gitrevisions(7)), but typically at least 'upstream' will be a branch
    from a remote repository.  Below are some examples:

    When 'downstream' is a local branch of a remote 'upstream', then just
    clone the upstream repo and set ``abc.upstream`` to whatever the
    master branch is and ``abc.downstream`` to whatever the local branch
    is, e.g. 'master' and 'downstream'::

      $ git clone $UPSTREAM_URL
      $ cd $REPO
      $ git checkout -b downstream
      $ git config abc.upstream master
      $ git config abc.downstream downstream

    When 'downstream' has its own remote repo, then either it or the
    upstream repo may be cloned (or neither, when cloning a third repo).
    The non-cloned repos must be added as remotes.  The example below
    clones 'downstream' and names the upstream remote 'upstream'.  For
    both the upstream and downstream repos, the master branch is named
    'master'::

      $ git clone $DOWNSTREAM_URL
      $ cd $REPO
      $ git config abc.downstream origin/master
      $ git remote add -f upstream $UPSTREAM_URL
      $ git config abc.upstream upstream/master

    When using a third repo, then both 'downstream' and 'upstream' are
    remotes::

      $ cd $REPO
      $ git remote add -f downstream $DOWNSTREAM_URL
      $ git remote add -f upstream $UPSTREAM_URL
      $ git config abc.downstream downstream/master
      $ git config abc.upstream upstream/master

2.  If necessary, override the regular expressions used to identify
    interesting upstream commits and/or the upstream links in downstream
    commits.  By default, an interesting upstream commit is one that has
    'fix' in its subject (case-insensitive), is a revert, CC's stable, or
    identifies a commit that it fixes using the 'Fixes:' tag (i.e. an
    interesting upstream commit is a possible fix).  Upstream links are,
    by default, any that 'git-show' or 'git-cherry-pick -x' would
    generate.  Internally git-abc interprets the interesting commit
    regular expressions with 'egrep' and the upstream link regular
    expressions with 'sed -E'.  The later requires the commit hash to be
    the first match (meaning the regex for the hash should be in ()'s).
    Additionally, git-abc determines where to look using git pretty format
    place holders, e.g. %s for subject and %b for body (see git-log(1)
    PRETTY FORMATS).  More than one place to look, or more than one
    expression per place, may be specified with a double comma (,,)
    separated list of <place-holders>:<regex> pairs.  The default
    expressions are::

      abc.should-highlight = %s:fix|Fix|FIX|Revert,,%b:^[Cc][Cc]: *<?[Ss]table[@ ]|^[Ff]ixes: *[0-9a-f]
      abc.upstream-link = %b:^commit ([0-9a-f]{40})$,,%b:^\\(cherry picked from commit ([0-9a-f]{40})\\)$

3.  Determine which paths should be used to filter backport candidates,
    if not all files of the repo need to be considered.  The example
    below uses the kernel's MAINTAINERS file to determine which files
    should be considered for VirtIO::

      $ VIRTIO_PATHS=$(awk -F: '/VIRTIO/,/^$/ {if (/F:/) print$2}' MAINTAINERS)

4.  Use git-abc to find backport candidates.  The example below only
    considers VirtIO files and uses ``abc.upstream`` and ``abc.downstream``
    to determine the revision range::

      $ git abc find -- $VIRTIO_PATHS

5.  List the candidates found with the previous 'git-abc find' step
    (interesting commits are highlighted)::

      $ git abc list

6.  Quickly accept/reject anything easy to accept/reject using
    'git-abc accept' and 'git-abc reject'.  The example below accepts any
    commits with the subject prefix 'YES' and rejects any commits with
    the subject prefix 'NO'::

      $ git abc accept $(git abc list | awk '/YES:/ {print$1}')
      $ git abc reject $(git abc list | awk '/NO:/ {print$1}')

    When setting up git-abc for the first time on a downstream that has
    already been maintained for some time, then, assuming the downstream
    is up to date with the current upstream already, all candidates may be
    rejected after the first 'find'.  When 'git-abc find' is run again,
    after upstream has changed, only new candidates that should be
    reviewed will show up in the list.

7.  Start an interactive session to review each remaining candidate for
    selection or rejection.  Repeat this step until all candidates have
    been accepted or rejected, at which point 'git-abc list' will no
    longer have any output::

      $ git abc select
      ...
      $ git abc list
      $

8.  If any candidates were accepted during the previous steps, then they
    will now show up when listing pending commits.  If a commit set as
    pending is later rejected, then it may be changed to rejected with
    'git-abc reject'::

      $ git abc list --pending
      $ git abc reject $PENDING_COMMIT_NO_LONGER_WANTED

9.  Backport pending commits using your favorite backport workflow (see
    gitworkflows(7))::

      $ git checkout -b $NEW_TOPIC_BRANCH
      $ git cherry-pick -x ...
      ...
      $ git filter-branch ...
      ...
      $ git format-patch ...
      $ git send-email ...

10. After some time refresh the upstream and downstream branches/remotes
    and then check for new candidates (i.e. return to step 3).  Anything
    backported for step 9 will now show up as backported, anything still
    pending will remain in the pending list, and any new candidates will
    show up in the candidate list (step 5 above)::

      $ git fetch --all # refresh remotes
      # use git-pull to refresh local branches with upstreams
      $ git abc find -- $PATHS
      $ git abc list --backported  # list of previous backports
      $ git abc list --pending     # list of still pending backports
      $ git abc list               # list of new candidates

11. Continue repeating steps 3-10 for the lifetime of the downstream fork.

Using Namespaces
================

When a developer needs to manage commits for multiple path sets (e.g. both
VirtIO and VFIO), then keeping the commit lists separate simplifies the
reviewing and management.  This can be done by using a unique namespace
for each::

  $ ABC_NAMESPACE=abc-virtio git abc find -- $VIRTIO_PATHS
  $ ABC_NAMESPACE=abc-vfio   git abc find -- $VFIO_PATHS
  $ ABC_NAMESPACE=abc-virtio git abc list # list VirtIO candidates
  $ ABC_NAMESPACE=abc-vfio   git abc list # list VFIO candidates

Creating git aliases with the following template allows one to remove
command line clutter::

  abc-<path-set-name> = "!_anon() {                                 \
    ABC_NAMESPACE="abc-<path-set-name>"                             \
    ABC_SHOULD_HIGHLIGHT="<path-set-should-highlight>"              \
    ABC_UPSTREAM_LINK="<path-set-upstream-link>"                    \
    ABC_HUNT_CHERRIES="<true|false>"                                \
    ABC_TODO_PATH="<path-set-todo-path>"                            \
    ABC_UPSTREAM="<path-set-upstream>"                              \
    ABC_DOWNSTREAM="<path-set-downstream>"                          \
    ABC_PATHS="<path-set-paths>"|$(<path-set-path-finding-command>) \
    git-abc "$@";                                                   \
  }; _anon"

It doesn't matter what the '_anon' function is called, and it may be the
same for all aliases.  For example, the VirtIO alias may be::

  abc-paths = "!_anon() { \
    awk -F: '/'\"$1\"'/,/^$/ {if (/F:/) print$2}' MAINTAINERS; \
  }; _anon"

  abc-virtio = "!_anon() { \
    ABC_NAMESPACE="abc-virtio" \
    ABC_PATHS=$(git abc-paths 'VIRTIO') \
    git-abc "$@"; \
  }; _anon"

and for VFIO::

  abc-vfio = "!_anon() { \
    ABC_NAMESPACE="abc-vfio" \
    ABC_PATHS=$(git abc-paths 'VFIO DRIVER') \
    git-abc "$@"; \
  }; _anon"

With the above aliases, candidates for VirtIO are found and listed with::

  $ git abc-virtio find
  $ git abc-virtio list

and, for VFIO, they are found and listed with::

  $ git abc-vfio find
  $ git abc-vfio list

Additional Features
===================

There are additional features documented in the man page
('git help abc'), but not exhibited in the workflow above.  Those features
are mostly for ABC flag maintenance.  For example, 'git-abc export' and
'git-abc import' are for saving and restoring the ABC flags, and
'git-abc reset' deletes them.  'git-abc flag' enables the user to easily
[re]flag commits as necessary.

Two useful list commands that do not fit the workflow above are
'git-abc list --downstream-only' and 'git-abc list --backported-non-trivial'.
'list --downstream-only' lists all commits that are downstream, but do not
have an upstream counterpart.  Keep in mind though that 'git-abc list' is
only ever relevant to the searches done with 'git-abc find'.  That means
any listing, including 'list --downstream-only', will only show commits
touching files used to limit the search.  If all downstream-only commits
of the downstream repo are needed, then a search with no limiting paths
given must be done first (a namespace dedicated for this purpose should
probably be used).  'list --backported-non-trivial' lists all backports
that were not trivial cherry picks.  Again, the listing is only ever
relevant to the previous searches.

Note, if a commit shows up in 'git-abc list --downstream-only' that
shouldn't be there, then the commit message and upstream-link expression
should be checked to see why it wasn't automatically linked.  Also, keep
in mind that while a well formed upstream link may be there, it may be
pointing to a commit hash that is not in the specified upstream, i.e. it
was most likely backported from a different upstream.  The
'git-abc set-upstream' command may be used to fix these types of issues,
and the counterpart command 'git-abc get-upstream' is also available.

Support
=======

Please report bugs to Andrew Jones <drjones@redhat.com>
