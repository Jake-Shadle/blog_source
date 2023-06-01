+++
title = "The case for embedded Windows bindings"
template = "page.html"
date = 2023-04-21

[taxonomies]
tags = ["windows", "rust"]
categories = ["code"]
+++

## Introduction

This post is going to be about a difference of opinion that I have regarding what users want from bindings to Windows, and the current state of ecosystem and where it is headed. I had been planning to write this post later, but I've gotten enough push back in recent PRs that I'm doing it now so that I can point people here to the full reasoning behind why I've been proposing changes to crates that bind to Windows.

## Windows Rust bindings

First, lets take a look at the current state of the ecosystem around Rust bindings to Windows.

### [`winapi`](https://github.com/retep998/winapi-rs)

`winapi` is the original "standard" crate for providing bindings to Windows (well, other than hand rolling), and has served that role well for years. It has mostly seen small changes over versions to add bindings to more functions/APIs, but there was one major overhaul that is worth noting, the move to split the crate into distinct features in the [0.2 -> 0.3](https://github.com/retep998/winapi-rs/issues/316) migration. This was done mostly to mitigate the high compile times for `winapi` (eg. ["Ah well 2s is much better than the 30 today!"](https://github.com/retep998/winapi-rs/issues/316#issuecomment-239697740)). You would assume that compile times should be quite low for a bindings crate, but it is not free, which we'll dig into more throughout this post.

At the time of writing, `winapi` has not seen any release in over 3 years, and while many crates still use it for calling into Windows (we currently have 43 crates in our graph that depend on it), it is slowly being supplanted by `windows` and `windows-sys`.

### [`windows-sys`](https://crates.io/crates/windows-sys)

`windows-sys` came on to the scene late 2021 from Microsoft itself and has several distinct advantages over `winapi`. Being maintained is a definite plus, but mostly it had the advantage of essentially covering the entire Win32 API surface of Windows as captured in [win32metadata](https://github.com/microsoft/win32metadata), as opposed to `winapi` which was mostly? handwritten. It also includes bindings to the [Windows Driver Kit](https://en.wikipedia.org/wiki/Windows_Driver_Kit) which notably includes bindings to some [`ntapi`](http://undocumented.ntinternals.net/) functions.

While `windows-sys` covers all of Win32 unlike `winapi`, `winapi` does have bindings to [some](https://github.com/retep998/winapi-rs/blob/5b1829956ef645f3c2f8236ba18bb198ca4c2468/src/um/d3d12.rs#L2006-L2254) [COM](https://en.wikipedia.org/wiki/Component_Object_Model) interfaces, while in `windows-sys` all COM interfaces are just represented as type aliases to `*mut c_void`.

### [`windows`](https://crates.io/crates/windows)

The `windows` crate, also from Microsoft, is the higher level sibling of `windows-sys`.

First, it provides a more Rust-like interface over the raw bindings in `windows-sys`, for example [`CreateEventA`](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createeventa)

```rust
// windows-sys
::windows_targets::link!("kernel32.dll" "system" #[doc = "*Required features: `\"Win32_System_Threading\"`, `\"Win32_Foundation\"`, `\"Win32_Security\"`*"] fn CreateEventA(lpeventattributes : *const super::super::Security:: SECURITY_ATTRIBUTES, bmanualreset : super::super::Foundation:: BOOL, binitialstate : super::super::Foundation:: BOOL, lpname : ::windows_sys::core::PCSTR) -> super::super::Foundation:: HANDLE);
```

```rust
// windows
#[doc = "*Required features: `\"Win32_System_Threading\"`, `\"Win32_Foundation\"`, `\"Win32_Security\"`*"]
#[cfg(all(feature = "Win32_Foundation", feature = "Win32_Security"))]
#[inline]
pub unsafe fn CreateEventA<P0, P1, P2>(lpeventattributes: ::core::option::Option<*const super::super::Security::SECURITY_ATTRIBUTES>, bmanualreset: P0, binitialstate: P1, lpname: P2) -> ::windows::core::Result<super::super::Foundation::HANDLE>
where
    P0: ::windows::core::IntoParam<super::super::Foundation::BOOL>,
    P1: ::windows::core::IntoParam<super::super::Foundation::BOOL>,
    P2: ::windows::core::IntoParam<::windows::core::PCSTR>,
{
    ::windows_targets::link!("kernel32.dll" "system" fn CreateEventA(lpeventattributes : *const super::super::Security:: SECURITY_ATTRIBUTES, bmanualreset : super::super::Foundation:: BOOL, binitialstate : super::super::Foundation:: BOOL, lpname : ::windows::core::PCSTR) -> super::super::Foundation:: HANDLE);
    let result__ = CreateEventA(::core::mem::transmute(lpeventattributes.unwrap_or(::std::ptr::null())), bmanualreset.into_param().abi(), binitialstate.into_param().abi(), lpname.into_param().abi());
    ::windows::imp::then(!result__.is_invalid(), || result__).ok_or_else(::windows::core::Error::from_win32)
}
```

In addition to the higher level wrappers around every function, the biggest difference is that `windows` includes bindings to all COM interfaces and WinRT interfaces/classes. This includes both the definitions of the virtual function tables at the heart of COM/WinRT, as well as wrapper methods on the interface to call into the correct vtable entry and return the result, which looks something like this.

```rust
#[repr(transparent)]
pub struct IModalWindow(::windows_core::IUnknown);
impl IModalWindow {
    pub unsafe fn Show<P0>(&self, hwndowner: P0) -> ::windows_core::Result<()>
    where
        P0: ::windows_core::IntoParam<super::super::Foundation::HWND>,
    {
        (::windows_core::Interface::vtable(self).Show)(::windows_core::Interface::as_raw(self), hwndowner.into_param().abi()).ok()
    }
}
::windows_core::imp::interface_hierarchy!(IModalWindow, ::windows_core::IUnknown);
unsafe impl ::windows_core::ComInterface for IModalWindow {
    const IID: ::windows_core::GUID = ::windows_core::GUID::from_u128(0xb4db1657_70d7_485e_8e3e_6fcb5a5c1802);
}
#[repr(C)]
pub struct IModalWindow_Vtbl {
    pub base__: ::windows_core::IUnknown_Vtbl,
    pub Show: unsafe extern "system" fn(this: *mut ::core::ffi::c_void, hwndowner: super::super::Foundation::HWND) -> ::windows_core::HRESULT,
}
```

## The difference of opinion

So roughly speaking, the view of the aforementioned crates is that they exist to provide complete (or in the `winapi` case, an as needed basis) bindings to Windows. If you want to call a Win32 function, or use a COM interface, you just add a dependency on the crate you want, enable the feature(s) you need for that functionality, and then use it in your code. Done!

But in my opinion....these crates aren't needed at all for a _vast majority_ of use cases, and in the case of `windows` and `windows-sys` actually have a negative impact on the Rust ecosystem.

Ok, that seems like a rather spicy take, let me explain further.

### What you get

External dependencies and their ease of use via cargo is without a doubt one of, if not the most, important reasons for the success of Rust. It's still rather magical to me (coming from C++ land) that adding a single line can just add an external chunk of code that is just (usually) ready to be used. No finagling with stupid build system integration or vendoring shenanigans or other nonsense.

Not only is adding a dependency easy, but updating it is as well, either for "free" if it is semver compatible, or (hopefully) minor changes in the code that depends on the external code to update it to a changed API. This ease of updating lets projects easily obtain bug fixes, performance improvements, new features etc.

Additionally, by using an external dependency, all the code inside of it can have its cost amortized across the number of crates that depend upon it, rather than potentially compiling multiple functionally similar/exact pieces of bespoke code.

### What it costs

As with many things though, external dependencies are not free.

The most notable cost of using an external crate over a local/vendored/inlined set of functionality is that you might be compiling code that you aren't actually using, and the likelihood of that only increases the larger the dependency is.

For example, say there is an HTTP client crate we want to use. The crate is very fully featured and supports HTTP 1, 2, and 3 and a bunch of other stuff. However, we only need to talk to HTTP/2 endpoints, so any code that is specific to the HTTP 1 or 3 support is costing us compile time for features that we aren't using, though hopefully that dead code is being correctly stripped out by the linker so end users aren't also taking the impact of increased binary sizes.

Cargo's mitigation for this problem are [features](https://doc.rust-lang.org/cargo/reference/features.html), which allow a crate to split up functionality into smaller chunks, so that crates can keep depending on the external crate, but only on the pieces of functionality that they actually want, rather than everything. This works fairly well for most crates...but not all.

### Windows is a breed apart

While the bindings crates I mentioned before all have features to narrow the scope of the crate, the truth is that the Win32 API is just too massive for features to work as they do for normal crates.

For some perspective, lets look at the [`windows-sys`](https://github.com/microsoft/windows-rs/commit/7e5cfb49895de30388a126c15a1bdf45d83268a0) Win32 API surface (we'll ignore the [WDK](https://en.wikipedia.org/wiki/Windows_Driver_Kit) even though it _is_ included in `windows-sys`):

* 213 features
* 16,778 extern functions
* 10,334 structs
* 951 unions
* 11,142 type aliases
* 114,452 constants

Now, in the `windows-sys` case, other than Copy + Clone being implemented for every struct and union (which is kind of something you don't want for structs that can be several hundred bytes large since it means it's too easy to accidentally do a full memory copy...), it does have the saving grace that there is no actual implementation, just type definitions and extern functions. But these still aren't free, and might take longer than you think to compile.

`windows` has the additional problem that, in addition to all of the types and constants in `windows-sys`, but every function in `windows-sys` present as a higher level wrapper around full of generics and actual implementation, but it also includes **6,845** COM interfaces, each of which has a bunch of implementation to present a friendly API over the lower level one exposed by the COM object.

That's a lot, but is it a problem?

Let's take a real world example from the primary project I work on, which depends on `windows-sys` (3 versions, we'll get to that), but no longer depends on `windows` since its compile times were far more egregious, but much easier to remove as we only depended on it via 3 transitive dependencies.

We'll look at just `windows-sys:0.45.0` as that is the version that takes the longest to compile due to having the most features enabled.

```ini
[
    Win32,
    Win32_Devices,
    Win32_Devices_HumanInterfaceDevice,
    Win32_Foundation,
    Win32_Globalization,
    Win32_Graphics,
    Win32_Graphics_Dwm,
    Win32_Graphics_Gdi,
    Win32_Media,
    Win32_NetworkManagement,
    Win32_NetworkManagement_IpHelper,
    Win32_Networking,
    Win32_Networking_WinSock,
    Win32_Security,
    Win32_Storage,
    Win32_Storage_FileSystem,
    Win32_System,
    Win32_System_Com,
    Win32_System_Com_StructuredStorage,
    Win32_System_Console,
    Win32_System_Diagnostics,
    Win32_System_Diagnostics_Debug,
    Win32_System_IO,
    Win32_System_Ioctl,
    Win32_System_Kernel,
    Win32_System_LibraryLoader,
    Win32_System_Memory,
    Win32_System_Ole,
    Win32_System_Performance,
    Win32_System_Pipes,
    Win32_System_SystemInformation,
    Win32_System_SystemServices,
    Win32_System_Threading,
    Win32_System_WindowsProgramming,
    Win32_UI,
    Win32_UI_Accessibility,
    Win32_UI_Controls,
    Win32_UI_HiDpi,
    Win32_UI_Input,
    Win32_UI_Input_Ime,
    Win32_UI_Input_KeyboardAndMouse,
    Win32_UI_Input_Pointer,
    Win32_UI_Input_Touch,
    Win32_UI_Shell,
    Win32_UI_TextServices,
    Win32_UI_WindowsAndMessaging
]
```

And....11.0s (2.2s codegen). Not bad! Except...this was on a 36 CPU Intel(R) Xeon(R) W-2295 CPU @ 3.00GHz machine, and seems, well, excessive.
