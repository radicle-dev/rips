---
RIP: 3
Title: Storage Layout
Author: '@fintohaps <fintan.halpenny@gmail.com>'
Status: Draft
Created: 2022-10-27
License: CC0-1.0
---

RIP #2: Storage Layout
======================

Introduction
------------

One of the key components of the Radicle network is the storage
layer. A key aspect of this component is that it is local-first. It
must be able to hold the local operator's view of a project as well as
the view of the peers that the operator is interested in.

Overview
--------

All nodes in the network must keep a local copy of the data they are
interested in. For local users this improves the experience of
interacting with the data -- not needing a server to compute data for
you and being able to work offline. This improves the availability of
data across the network.

The storage layer must also be designed for efficient replication of
data between peers in the network. For this reason, `git` is used as
the underlying database as it already efficiently exchanges data
between two machines ([Transfer Protocols][^0]).


With the above in mind, this RIP proposes a storage layer that
fulfills the following requirements:
1. It is able to keep a local copy of the working dataset.
2. It can store multiple resources -- specifically collaborative
   projects.
3. For each resource it can represent multiple peers' views of said
   resource.
4. It should natively interoperate with `git`.

Table of Contents
-----------------
* [Storage Home](#storage-home)
* [Layout](#layout)
* [Interoperability with `git`](#interoperability-with-git)
    * [Remote Replication](#remote-replication)
    * [Working Copy](#working-copy)
    * [Alternative Designs](#alternative-designs)
        * [Linking to a Working Copy](#linking-to-a-working-copy)
    * [Remote Helper](#remote-helper)
        * [Url](#url)
        * [Authorization](#authorization)
        * [Signing the Reference Set](#signing-the-reference-set)
* [Future Work](#future-work)
    * [Canonical References](#canonical-references)
* [Appendix](#appendix)
    * [Worked Example](#worked-example)
        * [Setup](#setup)
        * [Pushing Changes](#pushing-changes)
        * [Fetching Changes](#fetching-changes)
        * [Different Peers](#different-peers)
        * [Non-global Tags](#non-global-tags)
* [Credits](#credits)
* [Copyright](#copyright)

Storage Home
------------

It is trivial to store `git` repositories, locally, on disk. It is
also efficient to do so. The storage must be stored under the name
`storage` found under one of two base directories -- either the user's
home profile (as per the native operating system), or the `RAD_HOME`
environment variable.

If the `RAD_HOME` environment variable is set, the directory it points
to will be used as the base directory.

| Value                        | Example                       |
| ---------------------------- | ------------------------------|
| `$RAD_HOME/storage`          | /home/alice/MyRadicle/storage |

If `RAD_HOME` is not set, the directory `.radicle` will be used within
the native operating system's home directory is used, i.e. `HOME` on
Linux & macOS and `{FOLDERID_Profile}` on Windows.

|Platform | Value                                 | Example                         |
| ------- | ------------------------------------- | ------------------------------- |
| Linux   | `$HOME/.radicle/storage`              | /home/alice/.radicle/storage    |
| macOS   | `$HOME/.radicle/storage`              | /Users/Alice/.radicle/storage   |
| Windows | `{FOLDERID_Profile}\.radicle\storage` | C:\Users\Alice\.radicle\storage |

Layout
------

The layout must support multiple resources and multiple peers per
resource. Under the `storage` directory, each resource will be a [bare
git repository][^1]. For each resource in the storage, it must
have a stable and globally unique identifier. This is to ensure that
resources are stored uniquely in the `storage` and can be easily
addressed by their identifier.

```
storage/
  <resource>/   # Some project
    refs/       # All Git references
  <resource>/
  <resource>/
   …
```

For every resource, each peer associated with that resource must have
a separate, logical git repository -- which contains all the usual
reference namespaces, i.e. `heads`, `tags`, and `notes`.

This is to allow for peers to maintain different sets of changes for
the same piece of work, akin to `git` itself.

To have this separation, the [gitnamespaces][^2] feature of
`git` is used. For each peer, including the local operator, their
unique identifier is used as the namespace within the `<resource>`
repository. For now, this unique identifier is assumed to be a
hash-encoded value of the peer's public key, denoted `<pubkey>` from
here onwards.

This means that a peer's references will be scoped by
`refs/namespaces/<pubkey>`. We demonstrate this organisation below:

```
storage/
  <resource>/                      # Resource storage (bare repository)
    refs/                          # All Git references
      namespaces/                  # All replicated trees
        <pubkey>/                  # <pubkey>'s Git tree
          refs/
            heads/                 # <pubkey>'s branches
              master               # <pubkey>'s master branch
            tags/                  # <pubkey>'s tags
             …
        <pubkey>/
          refs/
            heads/
             …
           …
  <resource>/
    …
  <resource>/
    …
```

Note that top-level references may still exist,
i.e. `storage/<resource>/refs/{heads, tags}`. These namespaces must be
reserved for canoncial references -- references that are agreed,
collaboratively, as published and stable. How canonical references are
decided and written is left for further RIP.

```
storage/
  <resource>/                      # Resource storage (bare repository)
    refs/                          # All Git references
      HEAD                         # Canonical HEAD reference
      heads/
        master                     # Canonical version of master branch
      tags/
        v1.0.0                     # Canonical v1.0.0 release tag
       …
      namespaces/                  # All replicated trees
        <pubkey>/                  # <pubkey>'s Git tree
          refs/
            heads/                 # <pubkey>'s branches
              master               # <pubkey>'s master branch
            tags/
             …
        <pubkey>/
          refs/
            heads/
  <resource>/
    …
  <resource>/
    …
```

Interoperability with `git`
---------------------------

There are two aspects for interoperability with `git`:
1. Remote replication
2. Linking to a working copy

In the next sections we will cover how the above works with the
storage layout.

### Remote Replication

Remote replication pertains to fetching data from a remote peer. Since
the storage is a series of `git` repositories, the remote transfer of
data can be achieved via the `git` [protocols][^3]
and the correct [refspecs][^4]. Further detail is left to
another RIP, since the definition of the protocol use and the
cryptographic verification of fetched data is out of scope for this
RIP.

### Working Copy

A working copy is a local replica of the resource -- that is each
working copy corresponds to one resource in the storage -- where the
operator makes their own changes to the code base. This can be
compared to the way one would `git clone` from a mirrored repository,
e.g. GitHub, GitLab, etc. One can then make changes and push to the
mirror. In the case of Radicle, fetching and pushing changes is
between the working copy and the local-first storage.

The connection between the working copy and the storage is maintained
by a series of `git` [remotes][^5]. Each `git` remote
represents a single peer -- or namespace -- for that resource.

The name, i.e. `[remote."name"]`, is open to be defined by the
operator (or application), for example, the operator may use the
public key of the peer, `origin`, `rad`, a nickname `finto`, etc.

The `url` of the remote must be able to resolve the local storage's
resource corresponding to this working copy.

Since `gitnamespaces` are used, the `fetch` [refspec][^4] may
be:

```
fetch = +refs/heads/*:refs/remotes/<name>/*
```

The operator may also want to scope tags to particular remotes. This
can be achieved by using the `tagOpt` of a remote and adding another
fetch refspec.

```
fetch = +refs/tags/*:refs/remotes/<name>/tags/*
tagOpt = --no-tags
```

To `git fetch` or `git push`, it is necessary to state the namespace
that is being used for the operation. This can be achieved using `git
--namespace=<ns>` or `GIT_NAMESPACE=<ns> git`. Unfortunately, this
does not disallow the pushing to other peer's namespaces, nor allows
signing pushed references to one's own namespace. This is discussed in
[Remote Helper](#Remote-Helper).

To ensure `gitnamespaces` work as expected the author ran through a
worked example which is demonstrated in the [Appendix](#Appendix).

### Alternative Designs

An alternative design is to organise the peers under the `remotes`
namespaces, i.e. `refs/remotes/<pubkey>`. This particular namespace is
deemed special by `git` and its tooling. A "remote" reference is one
that corresponds to how a reference is fetched from a remote
location. The remote location and how to fetch/push from/to it is
configured using [`git remote`][^6]. When `git fetch` is used for
that remote, it will place the references under
[`refs/remotes`][^7].

#### Linking to a Working Copy

Continuing along this line of enquiry, we look at how this storage
will link to a working copy -- our personal directory for editing the
code. As we previously said, we will want to setup a remote in the
working copy. This will look like the following:

```
[remote "alice"]
url = file:///path/to/storage
fetch = +refs/remotes/<pubkey>/heads/*:refs/remotes/alice/*
```

This will do what you expect when running:

```
$ get fetch alice
```

However, you may be surprised that when running:

```
$ git fetch alice master
fatal: couldn't find remote ref master
```

It will not result in fetching the latest changes of `master`. In
fact, it will say no reference exists. To get the exact `master` we
are looking for we must run:

```
git fetch alice refs/remotes/alice/heads/master
```

To explain, `git` tends to work under a DWIM (Do What I Mean)
principle. The `master` in `git fetch alice master` is ambiguous, in
general. It could be `refs/heads/master`,
`refs/remotes/origin/master`, `refs/remotes/alice/heads/master`,
etc. `git` will assume that what you meant was `refs/heads/master` and
will look for this on the remote end, but of course it does not exist.

This problem is only compounded with [`refs/tags`][^7], where
pushing a tag to a remote will always DWIM and target the `refs/tags`
namespace -- unless otherwise specified.

### Remote Helper

In [Working Copy](#Working-Copy), it was discussed that there is no
way for a `git` remote to be configured so that it is aware of extra
logic, such as which `refs/namespaces` it should use (avoiding the
need for `--namespace`), disallow pushing to other peer's namespaces,
and signing references.

These three requirements can be solved by introducing a
`git-remote-rad` helper binary that can supply the namespace, ensure
the peer namespaces are respected, and that the signed references are
updated.

#### URL

The `url` scheme for a given remote is of the form:

```
rad://<resource>[/<pubkey>]
```

* The `rad://` scheme informs `git` to look for an executable
  `git-remote-rad` which will be executed during a `git-push` or
  `git-fetch`.
* The `<resource>` component is the resource identifier to be found in
  the storage.
* The `<pubkey>` is the namespace for which the the `--namespace`
  option will be set to. If `<pubkey>` is not specified, then the
  local operator's key is used.

#### Authorization

Since pushing to other peer's namespaces is disallowed, the value of
`<pubkey>` will be used to authorize whether a `git push` to a
particular `url` is allowed. If the `<pubkey>` of the `url` does not
match the local operator's key, then it must reject the push. Note
that no such authorization is needed for a `git fetch`.

#### Signing the Reference Set

To ensure that the local operator's reference set is cryptographically
verifiable, the reference set is signed with the operator's key during
a `git push` to their own namespace.

Future Work
-----------

### Canonical References

You may have noticed that in the new [layout](#Layout) the top-level
namespace is left for canonical references. The definition and
verification of canonicity is left for a future RIP.

Appendix
--------

### Worked Example

#### Setup

To begin we want to set up three git repositories: `storage`,
`project`, and `fork`. The `storage` repository will act like the
Radicle storage, while `project` and `fork` are working copies that
will be linked to `storage` via their remote entries.

```
# storage setup
$ mkdir storage
$ cd storage
$ git init --bare
```

```
# project setup
$ mkdir project
$ cd project
$ git init
```

```
# fork setup
$ mkdir fork
$ cd fork
$ git init
```

#### Pushing Changes

Our first action will be to make changes in `project` and push them to
`storage`. In order for us to do that we need to create a remote in
`project`, create a commit, and push it to `storage`.

```
# add remote, alice will mimic the public key hash
$ cd project
$ git remote add alice file:///home/user/RadicleTest/storage
```

```
# add a commit
$ touch README.md && git add README.md && git commit -am "Add README"
$ git --namespace=alice push alice master
```

`git` will then print out that it pushed a new branch and we can
confirm by inspecting the `refs` in `storage`.

```
# inspect refs
$ cd storage
$ tree refs
refs
├── heads
├── namespaces
│   └── alice
│       └── refs
│           └── heads
│               └── master
└── tags
```

#### Fetching Changes

Our next action will be to fetch the changes from `alice` in the `fork`
repository. To do this, we must add a remote -- like before -- and run
a `git fetch`.

```
# add remote, alice will mimic the public key hash
$ cd fork
$ git remote add alice file:///home/user/RadicleTest/storage
```

```
# fetch the changes
$ git --namespace=alice fetch alice
```

This will fetch the `heads` from `alice` and put them under the remote
`alice`. We can confirm this by inspecting the `refs` in `fork`.

```
# inspect refs
$ tree .git/refs
.git/refs
├── heads
├── remotes
│   └── alice
│       └── master
└── tags
```

#### Different Peers

To imitate the reality that there will be a namespace per peer, we add
a new remote for `fork`. We can then make changes to `alice/master` and
publish it under the `bob` namespace.

```
# add bob remote
$ git remote add bob file:///home/user/RadicleTest/storage
```

```
$ git merge bob/master
$ echo "Hello, Radicle" >> README.md
$ git commit -am "Hello, Radicle"
$ git --namespace=bob push bob master
```

Again, we can confirm this did what we wanted in `storage`.

```
# inspect storage refs
cd storage
tree refs
refs
├── heads
├── namespaces
│   ├── alice
│   │   └── refs
│   │       └── heads
│   │           └── master
│   └── bob
│       └── refs
│           └── heads
│               └── master
└── tags
```

#### Non-global Tags

Often we find that pushing tags pollutes the `refs/tags` namespace
since they do not get placed under `remotes` when fetching. With the
use of the `gitnamespaces` feature we avoid this.

```
$ cd fork
$ git tag v1.0.0
$ git push v1.0.0
```

```
# inspect storage refs
refs
├── heads
├── namespaces
│   ├── alice
│   │   └── refs
│   │       └── heads
│   │           └── master
│   └── bob
│       └── refs
│           ├── heads
│           │   └── master
│           └── tags
│               └── v1.0.0
└── tags
```

This shows that namespaces are superior in organising references
correctly for each given peer.

Credits
-------

* Kim Altintop -- for shining the light on the, lesser known,
  [gitnamespaces][^2] feature while developing `radicle-link`.
* Alex Good -- for attempting to implement a feature dubbed "ref
  rewriting" to solve the remotes problem, before realising that using
  [gitnamespaces][^2] could be a better option.

Copyright
---------

This document is licensed under the Creative Commons CC0 1.0 Universal license.

[^0]: https://git-scm.com/book/en/v2/Git-Internals-Transfer-Protocols
[^1]: https://git-scm.com/docs/git-init#Documentation/git-init.txt---bare
[^2]: https://git-scm.com/docs/gitnamespaces
[^3]: https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols
[^4]: https://git-scm.com/book/en/v2/Git-Internals-The-Refspec
[^5]: https://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes
[^7]: https://git-scm.com/book/en/v2/Git-Internals-Git-References
[^6]: https://git-scm.com/docs/git-remote
