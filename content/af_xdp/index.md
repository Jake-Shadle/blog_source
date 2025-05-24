+++
title = "`AF_XDP` sockets in Rust"
template = "page.html"
date = 2025-01-30

[taxonomies]
tags = ["rust", "linux", "networking"]
categories = ["code"]
+++

## Introduction

This post is about [e**X**press **D**ata **P**ath](https://en.wikipedia.org/wiki/Express_Data_Path), an [eBPF](https://en.wikipedia.org/wiki/EBPF) based network path available in Linux. Specifically, it is about [`AF_XDP`](https://en.wikipedia.org/wiki/Express_Data_Path#AF_XDP) sockets, which allow programs to take advantage of XDP features while in userspace

This post will go into details about XDP, eBPF, and `AF_XDP` sockets to hopefully give you a good idea of what they are and how to use them.

I also only cover Linux, though apparently eBPF and XDP/AF_XDP exist on Windows, even though I'm not entirely sure who Microsoft things their target audience is...

## Why `AF_XDP` sockets?



which allow packets to be sent and received from userspace with minimal interaction with the normal kernel networking stack, and supports features such as [DMA](https://en.wikipedia.org/wiki/Direct_memory_access) to further accelerate packet throughput.

I've been working on a [crate](https://github.com/Jake-Shadle/xdp) to provide `AF_XDP` socket functionality in Rust, specifically for [Quilkin](https://github.com/googleforgames/quilkin), which is a UDP proxy we use at Embark for our game servers. It currently uses [io-uring](https://en.wikipedia.org/wiki/Io_uring) for networking, but the goal of moving to eBPF/XDP was to reduce the compute resources needed for running the proxy, allowing us to support more sessions on the same hardware.

## eBPF

I'm not going to cover [eBPF](https://ebpf.io/what-is-ebpf/) in extensive detail since it's somewhat ancillary to the core of the post, but it's also a critical component of a network stack using `AF_XDP` sockets so I do need to go into some detail. Feel free to skip to the next section if you're already familiar with eBPF.

eBPF at its most basic is a way for userspace to provide the kernel with bytecode programs that the kernel validates, JITs, and attaches to a specific hook, based on the type of the bytecode program, invoking the bytecode program with the context provided by the hook when a hook event occurs. Due to these programs having such privileged access to networking/system calls/performance/etc events, loading eBPF programs always requires escalated privileges for the userspace program requesting for the eBPF program(s) to be loaded.

eBPF programs are essentially embedded programs. They are [`!#[no_std]`](https://docs.rust-embedded.org/book/intro/no-std.html) with no access to an allocator, and only have 512 bytes of stack space available, but they do have access to various [helpers](https://www.man7.org/linux/man-pages/man7/bpf-helpers.7.html) and [kfuncs](https://docs.ebpf.io/linux/kfuncs/) exposed by the kernel, based on the program type, for manipulating the context or data maps shared with userspace.

```txt

```

While there are many program types, they only one we need to care about is [`BPF_PROG_TYPE_XDP`](https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_XDP/). This program type can be loaded and attached to a network interface, and is invoked for _every_ packet received on that interface, with the full contents of that packet along with some associated metadata.

```rust
pub struct xdp_md {
    pub data: u32, // pointer to the beginning of the packet data
    pub data_end: u32, // pointer to the end of the packet data
    pub data_meta: u32, // pointer to a metadata section
    pub ingress_ifindex: u32,
    pub rx_queue_index: u32, // the queue index the NIC received the packet on
    pub egress_ifindex: u32,
}
```

Once invoked, an XDP program can read and/or write to the packet buffer, as long as it stays within the bounds of the memory buffer containing the packet, which is verified by the [verifier](https://docs.ebpf.io/linux/concepts/verifier/) before the kernel will load the program. The verifier is quite conservative since it must ensure that eBPF programs do not crash since that would tend to also crash the kernel. This is generally considered to be bad. If you thought Rust was annoying, the verifier will absolutely ruin your day.

Once your eBPF program has inspected/manipulated the packet, it needs to inform the kernel what to actually do with it when the eBPF program returns.

- `XDP_PASS` - This tells the kernel to continue passing the packet through the regular network stack to be handled as it would normally as if there was no eBPF program
- `XDP_TX` - This retransmits the packet back on the same network interface it was received on.
- `XDP_DROP` - Drops the packet completely, since XDP is so early in the stack, most traditional ways to monitor packet drops won't notice this.
- **`XDP_REDIRECT`** - This is the one we actually care about, it can be used to retransmit the packet on a different network interface, but for our purposes, it's how we can send a packet to an `AF_XDP` socket operating in userspace.

