# Backporting `rustc` in Ubuntu

This is a very informal guide detailing the process of backporting `rustc` in Ubuntu.

## Background

Every once in a while, old Ubuntu releases will need newer versions of `rustc`. [LP: #2100492](https://pad.lv/2100492) is a classic example- Firefox and Chromium started requiring Rust 1.82 to build, so every release going back to Focal needed the versioned `rustc-1.82` package in the archive.

The process of adding a new package to an old Ubuntu release is called a [backport](https://help.ubuntu.com/community/UbuntuBackports). Backports don't come with the same security guarantees as regular packages and must be manually enabled for a given machine.

Backporting Rust is tricky because every `rustc` package we create is designed to work with a specific version of the archive. It expects a particular version of LLVM, a particular version of CMake, a particular version of debhelper, etc... The challenge of backporting stems from trying to get the Rust package to work with an entirely different archive from the one it was built around.

## The Proper Order of a Backport

To backport `rustc`, we take a currently-working version and make it build on previous releases of Ubuntu. _However_, in order for a given `rustc` package to build, we _also_ need the previous _Rust_ version for that particular Ubuntu release.

### Example: Backporting `rustc-1.86` to Jammy

This concept is better illustrated by an example. Let's say you need to backport `rustc-1.86` to Jammy Jellyfish. In this example, here's the newest `rustc` version supported by every supported Ubuntu release:

| Ubuntu Release   | Supported `rustc` Version |
| ---------------- | ------------------------- |
| Jammy            | `rustc-1.83`              |
| Noble            | `rustc-1.83`              |
| Plucky           | `rustc-1.85`              |
| Questing (devel) | `rustc-1.86`              |

#### Going back in time, one step at a time

Trust me, you _don't_ want to try to backport the Questing version of `rustc-1.86` directly to Jammy. Since Plucky and Noble are just snapshots along the way from Jammy to Questing, you're not actually reducing the amount of work by jumping straight to Jammy.

Because of this, we're going to go "back in time" one step at a time: Questing -> Plucky, Plucky -> Noble, Noble -> Jammy. Doing it this way gives more immediate feedback and provides "checkpoints" along the way- if you have issues at, say, the "Noble -> Jammy" step, you _know_ that Noble works fine, so the issue stems from something Jammy-specific.

#### You need a bootstrap!

Remember, in order to build the Rust compiler, you need the _previous version's compiler_ to bootstrap it. Jammy only has `rustc-1.83` in this example. This means that we _also_ can't jump directly to `rustc-1.86` on Jammy- we have to backport `rustc-1.84` and `rustc-1.85` to Jammy as well!

#### Putting it all together

Now we know _what_ we have to do, AND the _order_ to do it in. To backport `rustc-1.86` to Jammy, we must perform the following backports in the given order:

1. `rustc-1.84`
   1. Backport `rustc-1.84` from Plucky to Noble
   2. Backport `rustc-1.84` from Noble to Jammy
2. `rustc-1.85`
   1. Backport `rustc-1.85` from Plucky to Noble
   2. Backport `rustc-1.85` from Noble to Jammy
3. `rustc-1.86`
   1. Backport `rustc-1.86` from Questing to Plucky
   2. Backport `rustc-1.86` from Plucky to Noble
   3. Backport `rustc-1.86` from Noble to Jammy

As you can see, something that seemed simple at first turns out to actually be a lot of work. (What else is new?) On the bright side, by doing things this way, you'll discover that the most common problems are the same, over and over again, and you'll eventually already know how to fix them in advance.

## Reference

From now on, `<X.Y>` and `<X.Y.Z>` refer to the Rust version number you're backporting.

`<X.Y_old>` and `<X.Y.Z_old>` refer to the Rust version number before the version you're backporting.

`<release>` refers to the Ubuntu release you're backporting _to_, while `<source_release>` refers to the Ubuntu release you're backporting _from_.

`<release_number>` is the version number of the Ubuntu release you're backporting to.

For example, if you're backporting `rustc-1.82` to Jammy...

- `<X.Y>` = 1.82
- `<X.Y.Z>` = 1.82.0
- `<X.Y_old>` = 1.81
- `<X.Y.Z_old>` = 1.81.0
- `<release>` = Jammy
- `<source_release>` = Noble
- `<release_number>` = 22.04

`<lpuser>` refers to your Launchpad username.

`<N>` is the suffix for `~bpo` in the [changelog version number](<backporting-rustc#3. Changelog Version>) and signals which crucial Rust dependencies (if any) were re-included in the source tarball.

`<lp_bug>` refers to the bug number on Launchpad.

## Setting up the Repository Locally

To set up a local repository, consult the [equivalent section](https://github.com/maxgmr/rust-maintenance-docs/blob/main/updating-rustc.md#setting-up-the-repository-locally) of the Rust toolchain update docs.

## The Backport Process

The baseline backport process is essentially trivial on its own and has few distinguishing features from a regular Rust toolchain update. The _real_ meat of these docs is the [Common Backporting Changes](<backporting-rustc#Common Backporting Changes>) section, which details things you'll often have to do in order to get the backport to build properly.

### 1. The Bug Report

To keep track of backport progress and status, a Launchpad bug report is absolutely necessary.

It's quite likely that there's a specific _reason_ why the backport was needed (e.g., a Rust-based application in an old Ubuntu release has an SRU that needs a newer toolchain to build). In this case, simply reference that bug report throughout the process, assigning the bug to yourself.

If no bug exists, you'll need to create your own. You can find a good example [here](https://pad.lv/2100492). If you need to go back multple Ubuntu releases, target the bug to _all_ series along the way as well, so each of the intermediate backports can be monitored. Additionally, if you need to go back multiple Rust versions, a separate bug report must be filed for each Rust version.

- Going back to our [Jammy 1.86 example](<backporting-rustc#Example: Backporting `rustc-1.86` to Jammy>), we'd have to create three bug reports:
  1. `rustc-1.84` bug targeting Noble and Jammy
  2. `rustc-1.85` bug targeting Noble and Jammy
  3. `rustc-1.86` bug targeting Plucky, Noble, and Jammy

### 2. Setup

Make sure you're on the `<X.Y>` branch of `<source_release>`, i.e. the version of Rust you want to backport on the Ubuntu release _newer_ than your target:

```shell
git checkout <source_release>-<X.Y>
```

Then, create your own branch:

```shell
git checkout -b <release>-<X.Y>
```

Example - backporting `rustc-1.85` to Jammy:

```shell
git checkout noble-1.85
git checkout -b jammy-1.85
```

### 3. Changelog Version

The first thing we should do on our new branch is create a new changelog entry right off the bat.

#### Version number format

The version number is complicated and takes the following general form:

```shell
<X.Y.Z>+dfsg0ubuntu1~bpo<N>-0ubuntu0.<release_number>
```

`<N>` signals the re-inclusion of crucial dependencies in the source tarball, and is one of the following three values:

- `0`: When `libgit2` _and_ LLVM are vendored
- `2`: When _only_ LLVM is vendored, and the `libgit2` from the archive is used
- `10`: When _only_ `libgit2` must be vendored, and the LLVM from the archive is used

If no extra vendoring of `libgit2` or LLVM was necessary, then the entire `~bpo<N>` portion of the version number can be excluded entirely.

Finally, in order to distinguish the in-progress backports from the final version, you should decrement `<release_number>` until you're ready to publish. For example, for a Noble backport, set the number to `24.03` until it's done, _then_ bump it to `24.04`.

Examples:

| Version Number                            | LLVM Vendored? | `libgit2` Vendored? | Ubuntu Release | Final Version? |
| ----------------------------------------- | -------------- | ------------------- | -------------- | -------------- |
| `1.85.1+dfsg0ubuntu1-0ubuntu0.24.04`      | No             | No                  | Noble          | Yes            |
| `1.80.0+dfsg0ubuntu1~bpo2-0ubuntu0.24.09` | Yes            | No                  | Oracular       | No             |
| `1.79.0+dfsg0ubuntu1~bpo0-0ubuntu0.22.03` | Yes            | Yes                 | Jammy          | No             |

#### Creating the new changelog entry

To begin, you only have to add/change `<release_number>` in the changelog version number. Don't forget to decrement it! You can leave any `~bpo<N>`s (or lack thereof) as-is for now, as you haven't made any changes to which dependencies have been vendored yet.

```shell
dch -bv \
    <old_version_number>.<release_number--> \
    --distribution "<release>"
```

Make the changelog entry description something like this:

```
  * Backport to <series> (LP: <lp_bug>)
```

### 4. Generating the Orig Tarball

Now, you need to grab the upstream source, with all the relevant files yanked out. `debian/watch` automates this process, so you just need to run `uscan` to download the source, yank out all the files listed in `Files-Excluded`, and create the repacked orig tarball:

```shell
uscan --download-version <X.Y.Z> -v 2>&1 | tee <path/to/log/file>
```

This process takes a while. If you're observant and/or experienced with this stuff, you'll notice that right now, _no changes_ have been made to the existing Ubuntu release's version. If you already have the proper orig tarball, there's no need to regenerate it right now. You _only_ need to regenerate the tarball when vendoring dependencies or otherwise changing the list of files excluded from the orig tarball.

At some point, we need to update `debian/watch` to match the `~bpo<N>` naming convention we've got right now. For now, however, you must sadly rename the tarball yourself to match the first portion of your changelog's version number:

```shell
mv \
    ../rustc-<X.Y>_<X.Y.Z>+dfsg1.orig.tar.xz \
    ../rustc-<X.Y>_<X.Y.Z>+dfsg0ubuntu1\~bpo<N>
```

- The `bpo~<N>` suffix helps keep track of your collection of orig tarballs! Looking at your big list of tarballs, you can tell at a glance _which_ versions have _which_ combinations of LLVM and/or `libgit2` included within them.

### 5. Attempting the Build

In _theory_, you may possibly be done at this step- you never know! In practice, this is when you see _where_ the build fails, so you can address the errors accordingly.

Consult the ["Local Build" section](https://github.com/maxgmr/rust-maintenance-docs/blob/main/updating-rustc.md#11-local-build) of the Rust toolchain update docs for info on how to test build locally using `sbuild`.

While something will almost _certainly_ break, most of these breakages are well-known. Consult the [Common Backporting Changes](<backporting-rustc#Common Backporting Changes>) section for assistance.

### 6. Building in a PPA

Once your backport builds locally, you're ready for a PPA upload!

You can follow the ["PPA Build" section](https://github.com/maxgmr/rust-maintenance-docs/blob/main/updating-rustc.md#12-ppa-build) of the Rust toolchain update docs for info on how to upload to a PPA.

### 7. Uploading the Backport

Once your backport builds successfully in a PPA for all targets, bump the `<release_number>` to its proper number and re-upload to your PPA once more.

After it builds, visit the `security-engineering` Mattermost channel and politely request they upload your backport. Make sure you include the following:

- A link to the bug report
- A link to the PPA

You can monitor upload progress in the [Security Proposed PPA](https://launchpad.net/~ubuntu-security-proposed/+archive/ubuntu/ppa/+packages?field.name_filter=rustc&field.status_filter=&field.series_filter=).

## Common Backporting Changes

### Vendoring LLVM

By default, `rustc` uses the distro's packaged LLVM instead of the vendored LLVM bundled in with the upstream Rust source.

However, if you see a message regarding `libclang-rt-*-dev`, `libclang-common-*-dev`, etc. not being installable, then the LLVM version in this Ubuntu release's archive is likely too old.

- Consult the Launchpad page for the relevant LLVM release to see if the right version is available. Example: [`llvm-toolchain-19`](https://pad.lv/u/llvm-toolchain-19) isn't available for Jammy. If the `rustc` version you're backporting uses LLVM 19 or newer, then in order to backport it to Jammy, you'll need to vendor.
- If you're not sure whether or not the "broken package" is part of LLVM, check to see which source package it belongs to: `dpkg -s <offending_package> | grep Source`

#### Re-including the upstream LLVM source

Since the Ubuntu Rust package doesn't typically need the vendored LLVM, we yank it out of the tarball. You need to _re-include_ the vendored LLVM source next time we generate the tarball. To prepare for this, remove `src/llvm-project` from `Files-Excluded` in `debian/copyright`:

```diff
@@ -4,7 +4,6 @@ Source: https://www.rust-lang.org
 Files-Excluded:
  .gitmodules
  *.min.js
- src/llvm-project
 # Pre-generated docs
  src/tools/rustfmt/docs
 # Fonts already in Debian, covered by d-0003-mdbook-strip-embedded-libs.patch
```

#### Modifying `d/control` and `d/control.in`

First, you'll need to remove the relevant packages from `Build-Depends` in both `d/control` AND `d/control.in`:

```diff
@@ -17,11 +17,7 @@ Build-Depends:
  python3:native,
  cargo-<X.Y_old> | cargo-<X.Y> <!pkg.rustc.dlstage0>,
  rustc-<X.Y_old> | rustc-<X.Y> <!pkg.rustc.dlstage0>,
- llvm-*-dev:native,
- llvm-*-tools:native,
- libclang-rt-*-dev (>= *),
- libclang-common-*-dev (>= *),
- cmake (>= *) | cmake3,
+ cmake (>= *) | cmake3 (>= *),
 # needed by some vendor crates
  pkgconf,
 # this is sometimes needed by rustc_llvm
@@ -54,7 +54,6 @@ Build-Depends:
  curl <pkg.rustc.dlstage0>,
  ca-certificates <pkg.rustc.dlstage0>,
 Build-Depends-Indep:
- clang-19:native,
  libssl-dev,
 Build-Conflicts: gdb-minimal (<< 8.1-0ubuntu6) <!nocheck>
 Standards-Version: 4.6.2
```

You'll also need to _add_ certain `Build-Depends` required to build LLVM:

```diff
@@ -37,6 +33,10 @@ Build-Depends:
  libgit2-dev (<< *),
  libhttp-parser-dev,
  libsqlite3-dev,
+# Required for llvm build
+ autotools-dev,
+ m4,
+ ninja-build,
 # test dependencies:
  binutils (>= *) <!nocheck> | binutils-* <!nocheck>,
  git <!nocheck>,
```

Finally, you can remove the binary package dependency as well:

```diff
@@ -271,7 +271,6 @@ Description: Rust formatting helper
 Package: rust-<X.Y>-all
 Architecture: all
 Depends: ${misc:Depends}, ${shlibs:Depends},
- llvm-*,
  rustc-<X.Y> (>= ${binary:Version}),
  rustfmt-<X.Y> (>= ${binary:Version}),
  rust-<X.Y>-clippy (>= ${binary:Version}),
```

#### Modifying `debian/config.toml.in`

Remove the option declaring LLVM as a dynamically-linked library (as opposed to the default statically-linked library):

```diff
--- a/debian/config.toml.in
+++ b/debian/config.toml.in
@@ -68,9 +68,6 @@ profiler = false

 )dnl

-[llvm]
-link-shared = true
-
 [rust]
 jemalloc = false
 optimize = MAKE_OPTIMISATIONS
```

The lines pointing `rustc` to the proper system LLVM tools can be removed in favour of using the default vendored LLVM:

```diff
--- a/debian/config.toml.in
+++ b/debian/config.toml.in
@@ -31,25 +31,6 @@ optimized-compiler-builtins = false
 [install]
 prefix = "/usr/lib/rust-RUST_VERSION"

-[target.DEB_BUILD_RUST_TYPE]
-llvm-config = "LLVM_DESTDIR/usr/lib/llvm-LLVM_VERSION/bin/llvm-config"
-linker = "DEB_BUILD_GNU_TYPE-gcc"
-PROFILER_PATH
-
-ifelse(DEB_BUILD_RUST_TYPE,DEB_HOST_RUST_TYPE,,
-[target.DEB_HOST_RUST_TYPE]
-llvm-config = "LLVM_DESTDIR/usr/lib/llvm-LLVM_VERSION/bin/llvm-config"
-linker = "DEB_HOST_GNU_TYPE-gcc"
-PROFILER_PATH
-
-)dnl
-ifelse(DEB_BUILD_RUST_TYPE,DEB_TARGET_RUST_TYPE,,DEB_HOST_RUST_TYPE,DEB_TARGET_RUST_TYPE,,
-[target.DEB_TARGET_RUST_TYPE]
-llvm-config = "LLVM_DESTDIR/usr/lib/llvm-LLVM_VERSION/bin/llvm-config"
-linker = "DEB_TARGET_GNU_TYPE-gcc"
-PROFILER_PATH
-
-)dnl
 [target.wasm32-wasi]
 wasi-root = "/usr"
 profiler = false
```

#### Modifying `debian/rules`

The build process must be modified somewhat in order to account for the newly-vendored LLVM.

```diff
--- a/debian/rules
+++ b/debian/rules
@@ -34,38 +34,37 @@ include debian/architecture.mk
 # for dh_install substitution variable
 export DEB_HOST_RUST_TYPE

+# Let rustbuild control whether LLVM is compiled with debug symbols, rather
+# than compiling with debug symbols unconditionally, which will fail on
+# 32-bit architectures
+CFLAGS := $(shell echo $(CFLAGS) | sed -e 's/\-g//')
+CXXFLAGS := $(shell echo $(CFLAGS) | sed -e 's/\-g//')
+
 # for dh_install substitution variable
 export RUST_LONG_VERSION
 export RUST_VERSION

 DEB_DESTDIR := $(CURDIR)/debian/tmp

-# Use system LLVM (comment out to use vendored LLVM)
-LLVM_VERSION = 19
-OLD_LLVM_VERSION = $(shell echo "$$(($(LLVM_VERSION)-1))")
-# used by the upstream profiler build script
-CLANG_RT_TRIPLE := $(shell llvm-config-$(LLVM_VERSION) --host-target)
-LLVM_PROFILER_RT_LIB = /usr/lib/clang/$(LLVM_VERSION)/lib/$(CLANG_RT_TRIPLE)/libclang_rt.profile
.a
-ifneq ($(wildcard $(LLVM_PROFILER_RT_LIB)),)
-# Clang per-target layout
-export LLVM_PROFILER_RT_LIB := /../../$(LLVM_PROFILER_RT_LIB)
-else
-# Clang legacy layout
-CLANG_RT_ARCH := $(shell echo '$(CLANG_RT_TRIPLE)' | cut -f1 -d-)
-ifeq ($(DEB_HOST_ARCH),armhf)
-CLANG_RT_ARCH := armhf
-endif
-export LLVM_PROFILER_RT_LIB := /usr/lib/clang/$(LLVM_VERSION)/lib/linux/libclang_rt.profile-$(CLANG_RT_ARCH).a
-endif
+# # Use system LLVM (comment out to use vendored LLVM)
+# LLVM_VERSION = 19
+# OLD_LLVM_VERSION = $(shell echo "$$(($(LLVM_VERSION)-1))")
+# # used by the upstream profiler build script
+# CLANG_RT_TRIPLE := $(shell llvm-config-$(LLVM_VERSION) --host-target)
+# LLVM_PROFILER_RT_LIB = /usr/lib/clang/$(LLVM_VERSION)/lib/$(CLANG_RT_TRIPLE)/libclang_rt.profi
le.a
+# ifneq ($(wildcard $(LLVM_PROFILER_RT_LIB)),)
+# # Clang per-target layout
+# export LLVM_PROFILER_RT_LIB := /../../$(LLVM_PROFILER_RT_LIB)
+# else
+# # Clang legacy layout
+# CLANG_RT_ARCH := $(shell echo '$(CLANG_RT_TRIPLE)' | cut -f1 -d-)
+# ifeq ($(DEB_HOST_ARCH),armhf)
+# CLANG_RT_ARCH := armhf
+# endif
+# export LLVM_PROFILER_RT_LIB := /usr/lib/clang/$(LLVM_VERSION)/lib/linux/libclang_rt.profile-$(
CLANG_RT_ARCH).a
+# endif
 # Cargo-specific flags
 export LIBSSH2_SYS_USE_PKG_CONFIG=1
-# Make it easier to test against a custom LLVM
-ifneq (,$(LLVM_DESTDIR))
-LLVM_LIBRARY_PATH := $(LLVM_DESTDIR)/usr/lib/$(DEB_HOST_MULTIARCH):$(LLVM_DESTDIR)/usr/lib
-LD_LIBRARY_PATH := $(if $(LD_LIBRARY_PATH),$(LD_LIBRARY_PATH):$(LLVM_LIBRARY_PATH),$(LLVM_LIBRAR
Y_PATH))
-export LD_LIBRARY_PATH
-endif
-
 ifneq (,$(filter parallel=%,$(DEB_BUILD_OPTIONS)))
 ifeq ($(DEB_HOST_ARCH),riscv64)
 NJOBS := -j $(patsubst parallel=%,%,$(filter parallel=%,$(DEB_BUILD_OPTIONS)))
```

It's no longer necessary to set certain configuration variables:

```diff
@@ -257,8 +256,6 @@ debian/config.toml: debian/config.toml.in debian/rules debian/preconfigure.st
amp
                -DDEB_TARGET_GNU_TYPE="$(DEB_TARGET_GNU_TYPE)" \
                -DMAKE_OPTIMISATIONS="$(MAKE_OPTIMISATIONS)" \
                -DVERBOSITY="$(VERBOSITY)" \
-               -DLLVM_DESTDIR="$(LLVM_DESTDIR)" \
-               -DLLVM_VERSION="$(LLVM_VERSION)" \
                -DRUST_BOOTSTRAP_DIR="$(RUST_BOOTSTRAP_DIR)" \
                -DRUST_VERSION="$(RUST_VERSION)" \
                -DPROFILER_PATH="profiler = \"$(LLVM_PROFILER_RT_LIB)\"" \
```

The `check-no-old-llvm` rule and certain other checks also become obsolete:

```diff
@@ -272,12 +269,7 @@ ifneq (,$(filter $(DEB_BUILD_ARCH), armhf armel i386 mips mipsel powerpc pow
erpc
        sed -i -e 's/^debuginfo-level = .*/debuginfo-level = 0/g' "$@"
 endif

-check-no-old-llvm:
-       # fail the build if we have any instances of OLD_LLVM_VERSION in debian, except for debian/changelog
-       ! grep --color=always -i '\(clang\|ll\(..\|d\)\)-\?$(subst .,\.,$(OLD_LLVM_VERSION))' --exclude=changelog --exclude=copyright --exclude='*.patch' --exclude-dir='.debhelper' -R debian
-.PHONY: check-no-old-llvm
-
-debian/dh_auto_configure.stamp: debian/config.toml check-no-old-llvm
+debian/dh_auto_configure.stamp: debian/config.toml
        # fail the build if the vendored sources info is out-of-date
        CARGO_VENDOR_DIR=$(CURDIR)/vendor /usr/share/cargo/bin/dh-cargo-vendored-sources
        # fail the build if we accidentally vendored openssl, indicates we pulled in unnecessary dependencies
@@ -365,13 +357,6 @@ ifneq (,$(filter $(DEB_BUILD_ARCH), armhf))
   FAILED_TESTS += | grep -v '^test \[debuginfo-gdb\] src/test/debuginfo/'
 endif
 override_dh_auto_test-arch:
-       # ensure that rustc_llvm is actually dynamically linked to libLLVM
-       set -e; find build/*/stage2/lib/rustlib/* -name '*rustc_llvm*.so' | \
-       while read x; do \
-               stat -c '%s %n' "$$x"; \
-               objdump -p "$$x" | grep -q "NEEDED.*LLVM"; \
-               test "$$(stat -c %s "$$x")" -lt 6000000; \
-       done
 ifeq (, $(filter nocheck,$(DEB_BUILD_PROFILES)))
 ifeq (, $(filter nocheck,$(DEB_BUILD_OPTIONS)))
        # there's a test that tests stage0 rustc, so we need to use system rustc to do that
```

Finally, we can clean up the LLVM source directory after installation to save on disk space:

```diff
@@ -507,6 +492,8 @@ endif
 override_dh_install-indep:
        dh_install
        $(RM) -rf $(SRC_CLEAN:%=debian/rust-$(RUST_VERSION)-src/src/usr/src/rustc-$(RUST_LONG_VERSION)/%)
+       # Get rid of src/llvm-project
+       $(RM) -rf debian/rust-$(RUST_VERSION)-src/usr/src/rustc-$(RUST_LONG_VERSION)/src/llvm-project
        # Get rid of lintian warnings
        find debian/rust-$(RUST_VERSION)-src/usr/src/rustc-$(RUST_LONG_VERSION) \
                \( -name .gitignore \
```

#### LLVM copyright stanza

We also need to re-include the LLVM copyright stanza in `debian/copyright`:

```diff
--- a/debian/copyright
+++ b/debian/copyright
@@ -919,6 +919,10 @@ Files-Excluded:
  vendor/zeroize_derive-1.4.2
 # DO NOT EDIT above, AUTOGENERATED

+Files: src/llvm-project/*
+Copyright: 2003-2025 University of Illinois at Urbana-Champaign
+License: Apache-2.0 with LLVM exception
+
 Files: C*.md
        R*.md
        Cargo.lock
```

#### Re-including the LLVM source

Update the [changelog version number accordingly](<backporting-rustc#3. Changelog Version>). Your version number should now contain either `~bpo0` or `~bpo2`, depending on the status of `libgit2`.

You can now [re-generate the orig tarball](<backporting-rustc#4. Generating the Orig Tarball>), which should now include the upstream LLVM source in `src/llvm-project`. Don't forget to also delete any existing `.debian.tar.xz` and `.dsc` files.

Grab all the new LLVM stuff and overlay it on your working directory:

```shell
cd ..
tar -xf rustc-<X.Y>_<X.Y.Z>+dfsg0ubuntu1\~bpo<N>.orig.tar.xz
cp -ra rustc-<X.Y.Z>-src/src/llvm-project rustc/src
```

Finally, you can add the vendored LLVM source to Git as well:

```shell
git add src/llvm-project
```

Annoyingly, some empty directories won't be included in the Git commit- [this is a problem shared by packages in general](https://pad.lv/1917877). Unfortunately, this means that you'll have to re-extract and overlay every time you clone the Git repo to a new place, run `git clean`, switch to a non-LLVM-vendored branch, etc.
