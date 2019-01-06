---
title: Let's Use dibs
---

# Let's Use [dibs][]!

[Dibs][dibs] (which stands for *Docker Image Build System*) makes it simple to
turn code into Docker images.

To some extent, it can be seen as an alternative to using a [Dockerfile][],
with the difference that [dibs][] provides finer control over the different
phases and makes it easier to land on a trimmed image.

A (hopefully!) gentle introduction can be found in its [README][dibs-readme];
this article focuses on a practical example of using Dibs for packaging an
external project that is not aware of Dibs itself (using the so-called [alien
mode][alien]).





More on this later...

[dibs]: https://github.com/polettix/dibs
[Dockerfile]: https://docs.docker.com/engine/reference/builder/
[dibs-readme]: https://github.com/polettix/dibs/blob/master/README.adoc
[alien]: https://github.com/polettix/dibs/blob/master/README.adoc#alien-mode
