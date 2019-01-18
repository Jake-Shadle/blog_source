+++
title = "Comparing Rust and C++ Evolution"
template = "page.html"
date = 2019-01-16

[taxonomies]
tags = ["rust", "cpp"]
categories = ["code"]
+++

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">In this week&#39;s <a href="https://twitter.com/cppcast?ref_src=twsrc%5Etfw">@cppcast</a> episode, Arthur made an offhand comment about C++ needing to move to fewer releases in order to ensure things are baked properly.<br><br>That&#39;s backward. You get proper bake time when you take away release pressure. Small frequent releases.</p>&mdash; Titus Winters (@TitusWinters) <a href="https://twitter.com/TitusWinters/status/1085219388779835397?ref_src=twsrc%5Etfw">January 15, 2019</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Some people I follow were commenting on this tweet, and it basically reminded me about several of the reasons why I believe in the future of Rust more than that of C++, so I figured I would lay them out in a longer format that hopefully will contain a bit more nuance than tweets.

## Scope

The first thing to do is clarify that when talking about Rust and C++, by evolution of "Rust" and "C++" I mean not only the language itself, but also the standard libraries for each. I realize that many C++ shops, particularly in game development, either don't use the standard library at all, because they have their own implementations of the things they actually need, or to such a small degree that they don't need, nor want, to care about how the standard library changes.

## How Changes Happen

C++ actually has a pretty decent summary of how to submit a proposal and the process required to get it into C++ [here](https://isocpp.org/std/submit-a-proposal). Summarized even more briefly:

1. Float idea and discuss in [mailing lists](http://lists.isocpp.org/mailman/listinfo.cgi)
1. Write a paper
1. Iterate on paper based on feedback
1. Present paper at a (physical) committee meeting
1. Paper gets accepted, rejected, or go back to 3
1. Wait for a new standard with your accepted proposal
1. Wait for all of the compilers/standard libraries you use to implement that part of the standard (or implement yourself in one of the OS compilers/libs)

Rust has the [Request For Comments (RFC) Process](https://github.com/rust-lang/rfcs/blob/master/README.md), which also briefly summarized is:

1. Float idea and discuss with people ([Probably here, but maybe not](https://internals.rust-lang.org/))
1. Write an RFC
1. Iterate on RFC based on feedback
1. Wait for the relevant Rust team to accept, reject, or postpone the RFC
1. Wait for the change to be implemented in the compiler/std lib, in the nightly compiler. (Or implement yourself of course!)
1. Iterate on the implementation, gather real feedback from other users/maintainers.
1. Either stabilize the change, or possibly even learn this was a "bad idea" and kill it fire.
1. Wait 6-12 weeks for the change to be moved into the stable compiler/std lib.
1. `rustup update stable` to get the latest stable release with the change

These processes are, broadly speaking, quite similar to each other, as they both follow an officially open process, but of course, the devil is in the details.

## Physical vs Virtual

While a significant portion of changing C++ can be done virtually, there does seem to be a hard requirement on there being at least one presentation of a change proposal at a physical committee meeting by someone who either wrote the paper, or is so familiar with the paper that the fact they didn't write it themselves is besides the point. And, as far as I'm aware, votes for determining the fate of a change proposal also have to be made in those physical meetings. This of course puts quite a steep barrier to entry for both those submitting proposals and the ultimate deciders upon those proposals, and is one that probably makes sense for C++ being a much more established language than many others. So while the C++ process is open, those physical restrictions means that the process is quite skewed towards large companies (Google, Microsoft, etc) or just generally those with time to travel

