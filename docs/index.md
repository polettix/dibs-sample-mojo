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
--- name: sample-mojo variables:
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


[dibs]: https://github.com/polettix/dibs
[Dockerfile]: https://docs.docker.com/engine/reference/builder/
[dibs-readme]: https://github.com/polettix/dibs/blob/master/README.adoc
[alien]: https://github.com/polettix/dibs/blob/master/README.adoc#alien-mode
[sample-mojo]: https://github.com/polettix/sample-mojo
[dokku-paas]: http://blog.polettix.it/dokku-your-tiny-paas/
[dibs-sample-mojo]: https://github.com/polettix/dibs-sample-mojo 
[dsm-dibs-yaml]: https://github.com/polettix/dibs-sample-mojo/blob/master/dibs.yml
