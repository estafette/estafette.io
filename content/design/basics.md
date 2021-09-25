---
title: "Basics"
description: "How does Estafette CI actually work?"
weight: 1
---

_Estafette CI_ leverages Kubernetes as a platform to run concurrent build jobs without being restricted by a set of pre-launched agents. This means it can run hundreds of builds or releases in parallel.

To actually execute build and release actions it takes a _pipeline-as-code_ approach by defining stages in an `.estafette.yaml` manifest file stored in an application's Git repository. These stages use (public) Docker container images to mount a shared working copy of your applications cloned repository into. This ensures that all changes happening in the working copy pass on to the next stage.

Having the ability to use public containers or hand-crafted containers makes it possible for a developer to control build-time dependencies themselves instead of having to add them to a pre-configured agent. Suddenly it's also possible to test out a new version of your popular framework by using a newer framework container in one of your branches. This makes it much more effortless to upgrade to the latest and greatest.

When splitting your build actions into multiple stages, there's a caveat that not all programming language package managers store their downloaded packages inside the working copy. _Node_ does in the `node_modules` directory, _Dotnet_ can do so by setting a packages location for the `dotnet restore` command and for _Java_ you can configure the Maven repository location to be inside the working copy. With this you can split multiple commands you want to run into separate stages so you gain visibility on the time spent in each of them and in case of failure see in one glance whether this happened during `restore`, `build` or `test`. However with other languages like _Golang_ you do not have control where the downloaded modules are stored, leading to every stage to download the modules again, wasting valuable time. In this case you're better of running multiple commands within the same stage.