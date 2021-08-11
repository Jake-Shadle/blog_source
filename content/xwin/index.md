+++
title = "Cross compiling Windows binaries from Linux"
template = "page.html"
date = 2021-08-06

[taxonomies]
tags = ["linux", "windows", "rust"]
categories = ["code"]
+++

## Introduction

Last November I added a new job to our CI to cross compile our project for `x86_64-pc-windows-msvc` from an `x86_64-unknown-linux-gnu` host. I had wanted to blog about this at the time but never got around to it, but after making some changes and improvements last month to this, in addition to writing a new utility, I figured now was as good of a time as any to share some knowledge in this area for those who might be interested.

## Why?

Before we get started with the [How](#how), I want to talk about why one might want to do this in the first place, as natively targetting Windows is a "known quantity" with the least amount of surprise. While there are reasons beyond the following, my primary use case for why I want to do cross compilation to Windows is our [Continuous Delivery](https://en.wikipedia.org/wiki/Continuous_delivery) pipeline for my main project at [Embark](https://www.embark.dev/).

### Speed

It's fairly common knowledge that, generally speaking, Linux is faster than Windows on equivalent hardware. From faster [file I/O](https://github.com/Microsoft/WSL/issues/873#issuecomment-425272829) to better utilization of high core count machines and faster process creation, many operations done in a typical CI job such as compilation and linking tend to be faster on Linux. And since I am lazy, I'll let another blog post about cross compiling [Firefox from Linux to Windows](https://glandium.org/blog/?p=4020) actually present some numbers in defense of this assertion.

### Cost

Though we're now running a Windows VM in our on-premise data center for our normal Windows CD jobs, we actually used to run it in [GCP](https://cloud.google.com/). It was **1** VM with a modest 32 CPU count, but the licensing costs (Windows Server is [licensed by core](https://www.microsoft.com/en-us/windows-server/pricing)) accounted for >20%! of our total costs for this particular GCP project, and that's not even counting the actual costs of the cores/RAM/disk for the VM.

<p align="center"><img src="graph_of_sadness.png"/></p>

### Containers

This one is probably the most subjective, so strap in!

While fast CI is a must, it really doesn't matter how fast it is if it gives unreliable results. Since I am the (mostly) sole maintainer (yah, we're trying to fix that!) for our CD pipeline in a team of almost 40 people, my goal was to get our pipeline into a reliably working state and easily maintain it with a minimal amount of my time, since I have other, more fun, things to do.

The primary way I did this was to build [buildite-jobify](https://github.com/EmbarkStudios/buildkite-jobify) (we use [Buildkite](https://buildkite.com/features) as our CI provider). This is just a small service that spawns [Kubernetes jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) for each of the CI jobs we run on Linux, based on configuration from the repo itself.

This has a few advantages and disadvantages over a more typical VM approach.

#### Pros

- Consistency - Every job run from the same container image 
- Clean builds - Since every job starts from a clean slate we don't have to worry about incremental compilation bugs.

#### Cons

- Clean builds - Clean builds are quite slow compared to incremental builds, however we mitigate this by using [sccache](https://github.com/mozilla/sccache).
- 

## How?

