Provide NSS modules globally, make nscd unnecessary (v2)

This is a follow-up to https://github.com/NixOS/nixpkgs/pull/138178 which fixes the binary incompatibilities of the original PR.

This PR allows glibc client binaries to access NSS modules configured via `system.nssModules` without nscd.
nscd has significant caching bugs and [causes friction](https://github.com/NixOS/nixpkgs/issues/95107) in general.
For details about nscd bugs, see issue [`DNS responses are cached`](https://github.com/NixOS/nixpkgs/issues/135888) and the [Fedora nscd deprecation notes](https://fedoraproject.org/wiki/Changes/DeprecateNSCD#Benefit_to_Fedora).

Some services set `LD_LIBRARY_PATH` to allow running them without nscd. These workarounds are now obsolete and are removed by this PR.

### Implementation

Provide global NSS modules at `/run/nss-modules-${word_size}-${glibc_version}/lib` (e.g. `/run/nss-modules-64-2.34/lib`) and patch glibc to use this path.
The versioning suffix ensures that only binary compatible glibc client binaries will use this path.

Repo [erikarvstedt/check-glibc-compatibilities](https://github.com/erikarvstedt/check-glibc-compatibilities/) shows that different NSS modules and glibc clients are compatible with each other, as long as they share the same minor glibc release (e.g. `2.34`).

Because the patched code region is never inlined, the patch affects all binaries that dynamically link glibc. This includes binaries prebuilt with a non-nixpkgs libc that are processed with `patchelf` (like `slack`).

### Todo
- nscd is still enabled to provide backwards compatibility for older binaries and 32-bit binaries on 64-bit hosts.
  In light of its defects and lack of maintenance, it might be sensible to disable nscd by default.
  Note: `unscd` is no replacement for nscd because it [doesn't implement](https://github.com/bytedance/unscd/blob/3a4df8de6723bc493e9cd94bb3e3fd831e48b8ca/nscd.c#L615-L621) all nsswitch functions ([src thread](https://github.com/NixOS/nixpkgs/pull/124019#issuecomment-856228888)).
- We'll add release notes for this PR as soon as it reaches community consensus.
- To support 32-bit binaries on 64-bit hosts without nscd, we could add options analogous to [`opengl.driSupport32Bit`](https://github.com/NixOS/nixpkgs/blob/610d4ea2750e064bf34b33fa38cb671edd893d3d/nixos/modules/hardware/opengl.nix#L60) and [`opengl.extraPackages32`](https://github.com/NixOS/nixpkgs/blob/610d4ea2750e064bf34b33fa38cb671edd893d3d/nixos/modules/hardware/opengl.nix#L99). As a minimum, `systemd` NSS modules should be provided. This can be addressed in another PR.

### Appendix
Fixes: #135888
Fixes: #105353
Cc: [#52411 (comment)](https://github.com/NixOS/nixpkgs/issues/52411#issuecomment-757347201)

This was long suggested in #55276.
