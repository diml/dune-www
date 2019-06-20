---
layout: blog
title: Plans for Dune 2.0.0
author: dimenix
tags: [ocaml, dune, jbuilder]
discuss: https://discuss.ocaml.org/t/plan-for-dune-2
---

The dune team is currently planning the 2.0.0 release of Dune and I
wanted to say a few words about our plans for it on how it will impact
Dune users. In this post, I will describe the main changes we are
planning to do and how this might impact users.

If you don't have the time to read the full posts, here are the
important points:

- dune 2.0.0 will be 99% backward compatible with dune 1.x
- jbuilder and dune 2.0.0 will be co-installable
- a dune-project file will now be necessary to mark the root of a project
- a few default behaviors will change in dune 2.0.0 but will not
  affect existing released packages
- dune 2.0.0 will require OCaml 4.08 to build, however it still be
  able to build projects using older compilers and will be installable
  in older opam switches

## Dropping support for Jbuilder

The main motivation for bumping the major version number is dropping
the support for jbuilder, as announced in previous posts:

- https://discuss.ocaml.org/t/dune-1-0-0-is-coming-soon-what-about-jbuilder-projects/2237
- https://dune.build/blog/second-step-deprecation/

This means that starting from Dune 2.0.0, the `dune` binary will no
longer be able to understand projects with `jbuild` files and will
even fail with seeing a `jbuild` file.

We are however making a slight change from our original plan. Right
now, `jbuilder` and `dune` are not co-installable, which is fine
because the `dune` package installs both `dune` and `jbuilder`
binaries and they can both still build existing jbuilder projects. We
initially planned that Dune 2.0.0 would continue to install a dummy
`jbuilder` binary except that it would fail on startup with an
informative message.

If we went ahead as planned, opam users would no longer be able to
install packages using jbiulder as the same time as packages using
dune 2.0.0. Given the large number of packages still using jbuilder,
this would effecitively split the opam package universe.

To avoid such a split, we are adapting our plans as follow: Dune 2.0.0
will no longer install a `jbuilder` binary, which means that Dune
2.0.0 and jbuilder will be coinstallable and in particular it will be
possible to install packages based on jbuilder at the same time as
packages requiring dune >= 2.0.0.

Since the last pure `jbuilder` packages in opam are now old and quite
far from the `jbuilder` binary distributed by dune, we will make one
last release of a `jbuilder` package based on the last feature release
of Dune 1.x.

### Mono-reposiotry users

In the end, the only users who will be affected are mono-repository
users since Dune 2.0.0 will not be able to understand a
mono-reposiotry that contains a mix of dune and jbuilder projects.

For these users, we recommend to run `dune upgrade` when importing
jbuilder projects in order to convert them to dune.

But really, we strongly encourage everybody maintaining a package
still using `jbuilder` to release a new version using `dune` :)
Upgrading from `jbuilder` to `dune` is extremely easy and automated as
desribed in [this post](https://dune.build/blog/second-step-deprecation/).

## 99% backward compatibility with Dune 1.x

When Dune sees `(lang dune x.y)` in a `dune-project` file, it adapts
itself to behave as the `x.y` versoin of Dune. This is how we provide
full backward compatibility in Dune.

This means that Dune 2.0.0 will not break anything. Projects currently
using `(lang dune 1.x)` will build with continue to build with all
versions of Dune from 1.x onwards.

There is one exception I can think of to this rule: currently if you
have a file that is both generated by a rule and present in the source
tree, this is a warning with older versions of the dune
language. However, making this a warning rather than an hard error is
costly and requires maintainig a complex piece of code in Dune.  As a
result, for this particular case we will make it an error in Dune 2.x,
which means that projects relying on it not build with Dune 2.x. FTR,
the way to fix this is to add a `(mode promote)` or `(mode fallback)`
field to the relevant stanza so that Dune knows what your intent is.

Appart from that, things that currently trigger a warning will
continue to trigger a warning when using `(lang dune 1.x)`, however
they'll become hard errors when using `(lang dune 2.x)`. We will also
change a few defaults behaviors in Dune 2.x.

You can thing of the `(lang dune x.y)` line in `dune-project` files as
follow: while working on a project, you should use the latest version
possible. This will gives you access to the most recent features of
Dune and will give you the most up-to-date commonly accepted defaults.

When not working on a project, this line is a way to remember which
version of Dune was used the last time you worked on the project so
that future version of Dune can continue to understand it without you
taking any action.

### Other changes in Dune 2.0.0

We are planning on making a couple of adjustments in Dune 2.0.0 to
keep it modern and up to date.  These shoudln't affect exising
released packages, however they might your workflow.

The most noteable one is the way dune detects the root of a
project. Currently there are two ways of marking the root of a
project:
- with a `dune-project` file
- by the presence of at least one `<package>.opam` file

The second rule is a bit arbitrary and only exist for historical
reason. We are going to remove it in Dune 2.x.

## Preparing the ground for multicore OCaml

Dune is a project that can clearly benefit from multicore. However, if
we continue with our current plan, Dune will effectively never be able
to benefit from multicore OCaml. Why is that? It is simply because
Dune is compatible with all versions of OCaml since 4.02.3.

After having asked the OCaml and Reason community about OCaml 4.02.3,
it's pretty clear to us that dropping support for 4.02.3 in Dune would
make life harder for quite a lot of people. So we want to keep
supporting OCaml 4.02.3.

However, while supporting building projects using OCaml 4.02.3 is
important, building Dune itself using OCaml 4.02.3 is not that
important. Currently, the version of OCaml Dune can build projects
against and the version of OCaml required to build Dune itself are
synchronised. This is what we want to change.

This is currently not supported by opam, however we have several ideas
to make it work. For instance, one idea is too simply create a
`ocaml408` package that would install a 4.08 compiler in a
sub-directory of the opam switch. In older opam switches, Dune would
use this package to build itself.

With this method, Dune 2.x would be installable in both old and new
opam switches, it would simply take longer to install in older
switches which seems acceptable.

In order to make all this work, we are going to make the `dune`
pakcage be a pure binary. Currently this package contains the
`dune.configurator` library which needs 
