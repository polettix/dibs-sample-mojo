---
---

[Dibs][dibs] (which stands for *Docker Image Build System*) makes it simple to
turn code into Docker images.

To some extent, it can be seen as an alternative to using a [Dockerfile][],
with the difference that [dibs][] provides finer control over the different
phases and makes it easier to land on a trimmed image.

A (hopefully!) gentle introduction can be found in its [README][dibs-readme];
this article focuses on a practical example of using Dibs for packaging an
external project that is not aware of Dibs itself (using the so-called [alien
mode][alien]).

Table of contents:

* TOC
{:toc}

# What Do We Want To Do?

In this article, we take a very simple web application from about two years ago
([sample-mojo][], already used [here][dokku-paas]) and define a [dibs][]
project to package it in a Docker image.

To do that, we have to:

- install the dependencies using [cpanm][] with the `cpanfile`
- select the artifacts thare of use in the final image
- produce an image with as little moving parts as possible, i.e. a [perl][]
  interpreter and the artifacts we selected.

To trim the image as much as possible, we use two phases: one for building the
artifacts and select the ones we need, another one to generate the image we are
after.

All of this will be done trying to minimize the amount of duplication of
configurations. Let's start!

# What Do We Need?

The original project we want to package is [sample-mojo][] and, being from
about two years ago, knows nothing about [dibs][]. No worries! We will setup a
*project directory* and work in *alien mode*, that is designed exactly for this
goal.

Our project directory is available at [dibs-sample-mojo][] and it is
essentially composed of the following files/directories:

    dibs.yml
    pack/
        dibsignore
        prereqs/
            alpine.build
            alpine.bundle

File `dibs.yml` is the main configuration file for Dibs, containing the
definition of the remote programs we need and the actions to perform; most
of this article is about this file actually.

File `pack/dibsignore` is similar in shape and semantics to a `.gitignore`
file; it is used by a remote pack to exclude the files we don't need in
the final image, as we will see.

Last, files in `pack/prereqs` define pre-requisites at the OS level for
the different phases of our image construction process (i.e. `build` and
`bundle`).

# The Configuration File

The whole configuration file (available [here][dsm-dibs-yaml]) is as
follows:

~~~ yaml
---
name: sample-mojo
variables:
   - &base     'alpine:3.6'
   - &builder  'sample-mojo-builder:1.0'
   - &bundler  'sample-mojo-bundler:1.0'
   - &target   'sample-mojo:1.0'
   - &username 'urist'
   - &appsrc   {path_src: '.'}
   - &appcache {path_cache: 'perl-app'}
   - &appdir   '/app'
origin: 'https://github.com/polettix/sample-mojo.git'
packs:
   basic:
      type: git
      origin: https://github.com/polettix/dibspack-basic.git
actions:
   ensure-prereqs:
      pack: basic
      path: prereqs
      args: ['-w', {path_pack: '.'}]
      user: root
   add-normal-user:
      pack: basic
      path: wrapexec/suexec
      args: ['-u', *username, '-h', *appdir]
      user: root
   base-layers:
      - from: *base
      - add-normal-user
      - ensure-prereqs
   buildish:
      envile:
         DIBS_PREREQS: build
   builder:
      extends: buildish
      actions:
         - base-layers
         - tags: *builder
   build-operations:
      - name: compile modules
        pack: basic
        path: perl/build
        user: *username
      - name: copy needed artifacts in cache
        pack: basic
        path: install/with-dibsignore
        args: ['--src', *appsrc,
               '--dst', *appcache,
               '--dibsignore', {path_pack: 'dibsignore'}]
        user: root
   build:
      extends: buildish
      actions:
         - from: *builder
         - ensure-prereqs
         - build-operations
   buildq:
      - from: *builder
      - build-operations
   bundlish:
      envile:
         DIBS_PREREQS: bundle
   bundler:
      extends: bundlish
      actions:
         - base-layers
         - tags: *bundler
   bundle-operations:
      - name: move artifacts in place
        pack: basic
        path: install/plain-copy
        args: [*appcache, *appdir]
        user: root
      - name: setup Procfile
        pack: basic
        path: procfile/add
        user: root
        env:
           PORT: 56789
        commit:
           entrypoint: ['/procfilerun']
           cmd: []
           user: *username
           workdir: *appdir
      - tags: *target
   bundle:
      extends: bundlish
      actions:
         - from: *bundler
         - ensure-prereqs
         - bundle-operations
   bundleq:
      - from: *bundler
      - bundle-operations
~~~

The `name` section just provides the name for temporary intermediate
container images. Parameter `origin` is set to the remote address of the
`git` repository of the code we want to package, which will be cloned and
checked out automatically inside sub-directory `src`. The rest is
explained in the following sections.

## Variables

As with regular code, it's useful to define some values once and then
reuse them multiple times. Dibs calls them *variables*, although they are
probably more like *constants*.

In our case, the `variables` section is the following:

~~~ yaml
variables:
   - &base     'alpine:3.6'
   - &builder  'sample-mojo-builder:1.0'
   - &bundler  'sample-mojo-bundler:1.0'
   - &target   'sample-mojo:1.0'
   - &username 'urist'
   - &appsrc   {path_src: '.'}
   - &appcache {path_cache: 'perl-app'}
   - &appdir   '/app'
~~~

All items are associated to a YAML anchor, for easy internal referencing
through aliases.

The first four items specify image names and tags, the first one being the
starting point (anchor `&base`), the two folling ones representing base
images for building and bundling, and the last one (anchor `&target`)
represeting our target image name. We will define two intermediate
container images in order to efficiently support a build phase (leveraging
an image with build tools inside) and a bundle phase (including only
runtime tools).

The build step, i.e. when Perl modules are compiled and installed, will be
performed using an unprivileged user account; anchor `&username` defines
the name for such account, and will be used both to create the user and to
run the build of modules, as well as it is set as the default user for
running the target container. Defining its value in one single place will
allow us to select some other username (e.g. one that is *not* tied to
game Dwarf Fortress!).

The last three items define paths. As anticipated, we will generate
artifacts in a build phase, at the end of which we select the needed ones
and copy somewhere inside the *cache*; afterwards, we will run the leaner
*bundle* container and put files in place, copying them *from* the cache.

## Pack(s)

This example makes use of a single dibspack, i.e. [dibspack-basic][],
because it contains all the programs that we need to run.

~~~ yaml
packs:
   basic:
      type: git
      origin: https://github.com/polettix/dibspack-basic.git
~~~

We assign name `basic`, so this is how we will refer to it in the rest of
the configuration.

## Actions

The `actions` section is by far the most crowded in the configuration
file.

### Pure Strokes

Actions `ensure-prereqs` and `add-normal-user` are pure *strokes*, defined
for convenience. Strictly speaking, `add-normal-user` is only used once
(inside `base-layers`) and might be *inlined*; on the other hand,
`ensure-prereqs` is used multiple times (in `base-layers` as well as in
`build` and `bundle`), so it's better to define it once and the use many
times.

~~~ yaml
actions:
   ensure-prereqs:
      pack: basic
      path: prereqs
      args: ['-w', {path_pack: '.'}]
      user: root
   add-normal-user:
      pack: basic
      path: wrapexec/suexec
      args: ['-u', *username, '-h', *appdir]
      user: root
   # ...
~~~

### "Abstract" Actions

~~~ yaml
actions:
   # ...
   buildish:
      envile:
         DIBS_PREREQS: build
   # ...
   bundlish:
      envile:
         DIBS_PREREQS: bundle
~~~

Actions `buildish` and `bundlish` are not real, full-fleshed actions. They
only serve the purpose of defining some basic characteristics (in this
case, the definition of an envile variable, i.e. a variable that is saved
as a file to avoid cluttering the environment) and not an real action.
This allows inheriting those characteristics from other actions (`builder`
and `build` both inherit from `buildish`, as well as `bundler` and
`bundle` inherit from `bundlish`). In the specific case, the
`DIBS_PREREQS` envile sets the `step` for action `ensure-prereqs` and will
eventually allow selecting between file `pack/prereqs/alpine.build` and
`pack/prereqs/alpine.bundle`.

### Base Images

Action `base-layers` includes all the actions to generate base images for
building and bundling, which is why is included as an action inside
`builder` and `bundler`.

~~~ yaml
actions:
   # ...
   base-layers:
      - from: *base
      - add-normal-user
      - ensure-prereqs
   # ...
   builder:
      extends: buildish
      actions:
         - base-layers
         - tags: *builder
   # ...
   bundler:
      extends: bundlish
      actions:
         - base-layers
         - tags: *bundler
   # ...
~~~

Actions `builder` and `bundler` take care to generate the base images for
respectively doing the build and bundle actions. For this reason,
`builder`'s last action is a frame to save image `sample-mojo-builder:1.0`
(as per alias `*builder`) which is then used as a base image by `build`
and `buildq`. Same applies to `bundler`/`bundle`/`bundleq`.

### Build Actions

The two main steps of building - i.e. compiling modules and selecting the
needed artifacts and copying them in the cache - are defined in
`build-operations`. This allows putting those two steps in both `build`
and `buildq`.

~~~ yaml
actions:
   # ...
   build-operations:
      - name: compile modules
        pack: basic
        path: perl/build
        user: *username
      - name: copy needed artifacts in cache
        pack: basic
        path: install/with-dibsignore
        args: ['--src', *appsrc,
               '--dst', *appcache,
               '--dibsignore', {path_pack: 'dibsignore'}]
        user: root
   build:
      extends: buildish
      actions:
         - from: *builder
         - ensure-prereqs
         - build-operations
   buildq:
      - from: *builder
      - build-operations
   # ...
~~~

Action `build` has a *quick* counterpart `buildq` that basically skip
action `ensure-prereqs`. This is an optimization: considering that
OS-level prerequisites will change rarely, most of the times we can be
sure that whatever was installed in the base image `builder` already
contains whatever we need.

Most strokes are executed as `root`, with the exception of modules
compilation that is executed as `urist` (i.e. the non-privileged
username). This user is available inside both the builder and the bundler
images thanks to step `add-normal-user`. Compiling with non-privileged
users is often a good security measure.

Last, it should be noted that neither `build` nor `buildq` results in an
image: the build phase is only instrumental to producing and isolating the
artifacts for the final bundle, so we can throw the containers away after
they have done their job.

### Bundle Actions

Most considerations for build actions also apply to bundling, considering
that they have the same structure, so they will not be repeated here.

The two main actions for *bundling* are collected in `bundle-operations`,
and consist in installing the cached artifacts in place (i.e. inside
`/app`) and then adding the program that interprets `Procfile`s.

~~~ yaml
actions:
   # ...
   bundle-operations:
      - name: move artifacts in place
        pack: basic
        path: install/plain-copy
        args: [*appcache, *appdir]
        user: root
      - name: setup Procfile
        pack: basic
        path: procfile/add
        user: root
        env:
           PORT: 56789
        commit:
           entrypoint: ['/procfilerun']
           cmd: []
           user: *username
           workdir: *appdir
      - tags: *target
   bundle:
      extends: bundlish
      actions:
         - from: *bundler
         - ensure-prereqs
         - bundle-operations
   bundleq:
      - from: *bundler
      - bundle-operations
~~~

Note that action for `setup Procfile` is different from other because
there are two additional keys:

- `env` sets an environment variable that will stick in the generated
  container image, providing a default value for a variable that is used
  in the `Procfile`;
- `commit` sets some characteristics of the image, e.g. the default user
  for running containers based on the image, the entry point, etc.

Differently from the build actions, both `bundle` and `bundleq` end up
saving the image, which is our target!

# Other files

Our project directory also include additional files that ease the build
and bundle phases.

## Prerequisites

Program `prereqs` in [dibspack-basic][] normally expects to find
a directory `prereqs` in the source code tree, containing scripts named
after OS distributions (e.g. `alpine`, `debian`, `ubuntu`, ...) and dibs
phases or *steps* (e.g. `build` or `bundle`).

In *alien mode*, though, we don't usually anticipate this in the source
code we want to package, so we have to provide our own scripts. They are
put inside `pack/prereqs` because `pack` is always mounted in the
containers generated by Dibs, and program `prereqs` expects to find the
right setup script in a sub-directory named `prereqs`.

This of course forces us to specify the base location of those setup
scripts when calling `prereqs`, which we do with argument `-w`:

~~~ yaml
actions:
   ensure-prereqs:
      pack: basic
      path: prereqs
      args: ['-w', {path_pack: '.'}]
      user: root
   # ...
~~~

You will note that we don't pass an explicit path (although it will
probably always be `/tmp/pack`), but use a construct that asks [dibs][] to
expand the path for us. In other terms, the associative array:

~~~
{path_pack: '.'}
~~~

is interpreted as a request to expand the path `.` relative to the mount
point of the `pack` directory inside the container.

In our case, the two prerequisite scripts are quite simple. This is the
one for build:

~~~
#!/bin/sh
apk --no-cache add build-base perl perl-dev
~~~

and this the one for bundle:

~~~
#!/bin/sh
apk --no-cache add perl
~~~

Of course we could have obtained pretty much the same result with an
immediate script, but this gives us the possibility to later expand our
project to produce images based on different distributions using the same
scaffolding.

## Files (De)Selection

The other file we added in the project is `pack/dibsignore`. Again, the
position in `pack` is because that directory is available inside the
container.

A *dibsignore* file is much like a *gitignore*, having the same syntax and
semantics. It is used by program `install/with-dibsignore` in
[dibspack-basic][] to filter out unwanted files/directories when selecting
artifacts (most probably, those that survive will be part of the final
image).

In our case, the file contains the following:

~~~
/.buildpacks
/cpanfile
/.git
~~~

i.e. we want to get everything apart file `.buildpacks` (which is used by
Dokku while building the image), file `cpanfile` (not needed after
compilation of modules) and directory `.git` (not needed in the final
image).

Dibsignore files are usually located in the source tree but, again, we are
in *alien mode* and the original code does not have them. This is why we
have to specify an absolute location in the configuration file:

~~~ yaml
actions:
   # ...
   build-operations:
      # ...
      - name: copy needed artifacts in cache
        pack: basic
        path: install/with-dibsignore
        args: ['--src', *appsrc,
               '--dst', *appcache,
               '--dibsignore', {path_pack: 'dibsignore'}]
        user: root

~~~

Again, we specify the path to the dibsignore file asking Dibs to expand it
for us, selecting file `dibsignore` inside the mount point for
sub-directory `pack`.

# So What Now?

If you were teased... great!

Installing [dibs][] is not difficult (clone the repo, run `carton` and set
`PERL5LIB`), but it will be easier shortly (using a Docker image). In the
meantime, *any* feedback (well, any *constructive* feedback) is more than
welcome!


[dibs]: https://github.com/polettix/dibs
[Dockerfile]: https://docs.docker.com/engine/reference/builder/
[dibs-readme]: https://github.com/polettix/dibs/blob/master/README.adoc
[alien]: https://github.com/polettix/dibs/blob/master/README.adoc#alien-mode
[sample-mojo]: https://github.com/polettix/sample-mojo
[dokku-paas]: http://blog.polettix.it/dokku-your-tiny-paas/
[dibs-sample-mojo]: https://github.com/polettix/dibs-sample-mojo 
[dsm-dibs-yaml]: https://github.com/polettix/dibs-sample-mojo/blob/master/dibs.yml
[perl]: https://www.perl.org/
[dibspack-basic]: https://github.com/polettix/dibspack-basic
