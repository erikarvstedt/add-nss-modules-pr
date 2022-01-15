Provide NSS modules globally, make nscd unnecessary (v2)

This is a follow-up to https://github.com/NixOS/nixpkgs/pull/138178 which fixes the binary incompatibilities of the original PR.

This PR allows glibc client binaries to access NSS modules configured via `system.nssModules` without nscd.
nscd has significant caching bugs and [causes friction](https://github.com/NixOS/nixpkgs/issues/95107) in general.
For details about nscd bugs, see issue [`DNS responses are cached`](https://github.com/NixOS/nixpkgs/issues/135888) and the [Fedora nscd deprecation notes](https://fedoraproject.org/wiki/Changes/DeprecateNSCD#Benefit_to_Fedora).

Some services set `LD_LIBRARY_PATH` to allow running them without nscd. These workarounds are now obsolete and are removed by this PR.

### Implementation

Provide global NSS modules at `/run/nss-modules${word_size}-${glibc_version}/lib` (e.g. `/run/nss-modules64-2.34/lib`) and patch glibc to use this path.
The versioning suffix ensures that only binary compatible glibc client binaries will use this path.

Repo [erikarvstedt/check-glibc-compatibilities](https://github.com/erikarvstedt/check-glibc-compatibilities/) shows that different NSS modules and glibc clients are compatible with each other, as long as they share the same minor glibc release (e.g. `2.34`).

### Todo
- nscd is still enabled to provide backwards compatibility for older binaries.
  In light of its defects and lack of maintenance, it might be sensible to disable nscd by default.
- We'll add release notes for this PR as soon as it reaches community consensus.

### Appendix
Fixes: #135888
Fixes: #105353
Cc: [#52411 (comment)](https://github.com/NixOS/nixpkgs/issues/52411#issuecomment-757347201)

This was long suggested in #55276.
