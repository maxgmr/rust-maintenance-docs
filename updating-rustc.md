# Updating `rustc` in Ubuntu

This is a very informal guide detailing the process of maintaining the Rust toolchain in Ubuntu.

## Background

We publish the `rustc` source package, which provides binary packages for the Rust toolchain, as a versioned package, meaning that a new source package is created for every Rust release (e.g., [`rustc-1.83`](https://pad.lv/u/rustc-1.83) and [`rustc-1.84`](https://pad.lv/u/rustc-1.84)).

These packages are maintained largely in order to support building other Rust packages in the Ubuntu archive. Rust developers seeking to work on their own Rust programs typically use the [`rustup` snap](https://snapcraft.io/rustup) instead.

Unlike most packages, we maintain `rustc` separately from Debian. Sometimes changes are merged, but by and large we maintain our Rust toolchain independently.

## High-Level Summary of the Update Process

A typical `rustc` update goes through the following steps:

1. The source code of the new Rust version is downloaded from upstream, overwriting the old upstream source code.
2. The existing package patches are refreshed so they apply properly onto the new Rust source code.
3. Unnecessary vendored dependencies are pruned.
4. The upstream Rust source is re-downloaded with the new list of files to yank out, overwriting the source code once again, but with the unnecessary files removed.
5. Any issues causing the new Rust package build to fail are fixed.

Usually, the process is _largely_ formulaic and straightforward until step 5- that's when you'll need to address any issues causing the build to fail.

## Reference

From now on, `<X.Y>` and `<X.Y.Z>` refer to the Rust version number of the version you're updating _to_. `<X.Y_old>` and `<X.Y.Z_old>` refer to the version you're updating _from_. As an example, let's say you were updating `rustc-1.84` (upstream `1.84.0`) to `rustc-1.85` (upstream `1.85.1`):

- `<X.Y>` = `1.85`
- `<X.Y.Z>` = `1.85.1`
- `<X.Y_old>` = `1.84`
- `<X.Y.Z_old>` = `1.84.0`

`<lpuser>` refers to your Launchpad username.

I'll refer to the different remotes by certain names; just substitute these values with whatever you decided to name the remotes.

- `<foundations>`: The Foundations [`rustc` Git repo](https://git.launchpad.net/~canonical-foundations/ubuntu/+source/rustc)
- `<lpuser>`: Your personal Launchpad Git repo. [Here's mine for reference!](https://git.launchpad.net/~maxgmr/ubuntu/+source/rustc/)

Unless stated otherwise, all file paths are relative to the Git repository directory, i.e., the directory containing the `debian/` directory.

When describing file paths, the `debian/` directory is sometimes shortened to `d/` for brevity. (Same with `debian/patches` being shortened to `d/p`.)

`<release>` refers to the target Ubuntu release adjective, e.g. `noble` for Noble Numbat.

## Setting up the Repository Locally

Personally, since the Debian build tools generate a bunch of files in the parent directory of your package source directory, I like to keep things two directories deep, i.e., clone the repo inside an existing `rustc` directory so the structure looks like `rustc/rustc/<debian dir and source files>`. This means that all your orig tarballs, `.changes` files, etc. will be inside the upper-level `rustc` folder.

Clone the Foundations [`rustc` Git repo](https://git.launchpad.net/~canonical-foundations/ubuntu/+source/rustc):

```shell
git clone git+ssh://<lpuser>@git.launchpad.net/~canonical-foundations/ubuntu/+source/rustc
```

Create your own personal Git repo on Launchpad:

```shell
git remote add <lpuser> git+ssh://<lpuser>@git.launchpad.net/~<lpuser>/ubuntu/+source/rustc
```

## The Update

### 1. Creating a Bug Report

In order to publicly track the progress and status of the update, you must create a bug report on Launchpad.

If this Rust version is the target version for the upcoming Ubuntu release, then you can create a bug under [`rust-defaults`](https://pad.lv/u/rust-defaults). This is because you will eventually need to update `rust-defaults` to point to this new Rust version. You can find a real-life bug report of mine for `rustc-1.85` [here](https://pad.lv/2109761).

If this Rust version is _not_ the target version for the upcoming Ubuntu release, then the process is more similar to adding a [new package](https://wiki.ubuntu.com/UbuntuDevelopment/NewPackages) to the archive in general. You can create an Ubuntu bug tagged with `needs-packaging`. A real-life bug report for `rustc-1.86` is [here](https://pad.lv/2117513).

Make sure to add a tag for your target Ubuntu release to your bug report, e.g. `noble` if Noble Numbat is the current devel release. You should also set the bug Milestone to the version number of the release, e.g. `ubuntu-24.04` for Noble Numbat.

### 2. Setting up

Make sure you're on the previous version's branch:

```shell
git fetch --all
git checkout merge-<X.Y_old>
```

Create your new branch, then upload it to your personal Git repo:

```shell
git checkout -b merge-<X.Y>
git push <lpuser> merge-<X.Y>
```

### 3. Getting the New Upstream Rust Source

In this step, we need to get the source code of the new Rust version. Luckily, `debian/watch` automates this process for us.

#### Removing the old vendor prune list

First, understand that we prune a great deal of unnecessary vendored crate dependencies from the `vendor/` directory.

We tell the Debian packaging tools which files from the Rust team's source code it can ignore in `debian/copyright`. The list of ignored vendored dependencies is huge- you'll automatically generate that list later. For now, we're just going to include _all_ vendored dependencies from the new Rust version to make sure we don't miss anything.

To do so, remove the `DO NOT EDIT (...) AUTOGENERATED` chunk from `d/copyright`:

```diff
- # DO NOT EDIT below, AUTOGENERATED
-  vendor/addr2line-0.17.0
-  vendor/aes-0.8.4
-  vendor/ahash-0.8.10
- (...)
-  vendor/zerocopy-derive-0.8.14
-  vendor/zeroize_derive-1.4.2
- # DO NOT EDIT above, AUTOGENERATED
```

#### Updating the changelog and package name

You can also update the changelog, manually setting the version number:

```shell
dch -v <X.Y.Z>+dfsg0ubuntu0-0ubuntu0
```

Don't forget to manually change the versioned package name in the changelog too! (i.e., `rustc-<X.Y_old>` -> `rustc-<X.Y>`)

You can also create your first changelog bullet point- the "New upstream version" point. It should look something like the following; consult previous changelog entries for examples:

```
* New upstream version <X.Y.Z> (LP: #<lp_bug_number>)
```

Make sure `<lp_bug_number>` matches the bug number you created earlier!

#### Getting the new source and orig tarball with `uscan`

We can use `uscan` (from the `devscripts` package) to get the new source code and generate the orig tarball:

- I like to save the log output somewhere because `uscan` will warn you if any files you've excluded in `d/copyright` aren't actually in the original source in the first place. You'll want to take a look at those at some point and make sure the exclusions aren't a typo or something.

```shell
uscan --download-version <X.Y.Z> -v 2>&1 | tee <path/to/log/output>
```

You may have to rename the orig tarball to match the first part of your package version number, i.e., `rustc-<X.Y>_<X.Y.Z>+dfsg0ubuntu0`.

#### Updating the source code in your repository

`uscan` just downloads the new Rust source, yanks out the ignored files, and packs the orig tarball. Your actual Git repository hasn't changed at all yet. To do that, we can use `gbp` to import the new Rust version onto your existing repository.

- This is the point in which your _actual source code_ moves from `<X.Y_old>` to `<X.Y>`.

We use the `experimental` branch to store the upstream sources. Make sure you reset this branch beforehand, branching it off from your current Git branch:

```shell
git branch -D experimental
git branch experimental
```

Now, we're ready to invoke `gbp`:

- `--no-symlink-orig`: Don't create a symlink from the upstream tarball to a Debian-policy-conformant upstream tarball name.
- `--no-pristine-tar`: Don't generate a `pristine-tar` delta file.
- `--upstream-branch=experimental`: Put the upstream source into the `experimental` branch of your Git repo.
- `--debian-branch=merge-<X.Y>`: Merge the new upstream version into your current branch.

```shell
gbp import-orig \
    --no-symlink-orig \
    --no-pristine-tar \
    --upstream-branch=experimental \
    --debian-branch=merge-<X.Y> \
    ../rustc-<X.Y>_<X.Y.Z>+dfsg0ubuntu0.orig.tar.xz
```

You should now see two commits in your Git log stating that your upstream source has been updated. Great job!

### 4. Initial Patch Refresh

Now that the actual upstream source code of your repo has changed, many of the different patches in `debian/patches` won't apply cleanly. It's your job to fix them.

In general, this is pretty straightforward- you just need to use `quilt` to find which patches need refreshing, then figure out how the patch changes apply to the updated source.

Some common changes include:

- A patch which applies to a versioned vendored dependency can't find the files it wants to patch (because the version number in the vendored dependency file path got updated). Find the updated vendored dependency and apply the changes _there_ instead.
- A patch is made obsolete because the changes were incorporated upstream. Great! You can drop the patch entirely (as long as you're _sure_ it's now unneeded).
- The surrounding context of a patch changed and now there's some fuzz. This is the most common and easiest situation, you just have to apply the exact same changes and refresh the patch.

To check your work, you can always run a `git diff` on the patch you just refreshed. If the only changes to the patch are to the surrounding _context_ or location of the changes, rather than the changes _themselves_, you've done a great job!

- The situations in which you have to modify the _actual changes_ made by the patch are relatively limited. Be extra-cautious in these cases.

### 5. Pruning Vendored Crates

As stated above, we don't want to include unnecessary vendored dependencies, especially Windows-related crates like `windows-sys`. This pruning ensures adherence to free software principles, reduces the attack surface of the binary packages, and reduces the binary package size on the end user's hard drive.

Since we removed the autogenerated `d/copyright` chunk earlier before grabbing the updated upstream source with `uscan`, our `vendor/` directory will contain _everything_. It's your job to remove the dependencies on things we don't need.

#### Get a list of things to prune

To assist in this, you can use [this script](https://github.com/canonical/foundations-sandbox/blob/master/maxgmr/win-rustc-prune-list) I created based on the comment at the top of `d/p/prune/d-0021-vendor-remove-windows-dependencies.patch`. It searches the `Cargo.toml`s of all the crates within `vendor/` for Windows-related terms, then outputs all the lines which contain those terms.

Make sure you push the existing pruning patch first, otherwise the list of potential prune candidates will be incorrect!

```
quilt push debian/patches/prune/d-0021-vendor-remove-windows-dependencies.patch
win-rustc-prune-list > path/to/prune-list
```

Note that the majority of this list will be taken up by lines from the `Cargo.toml`s of various versions of the Windows crates like `windows`, `windows-sys`, etc. You don't need to prune these! Basically, if you _know_ that a given crate will be removed, you _don't_ need to prune its `Cargo.toml`.

#### Prune vendored Windows dependencies

Your top priority will be removing any dependencies which pull in `windows-sys`, `winapi`, `ntapi`, `windows`, `schannel`, etc. Go through the prune list you just generated and inspect the flagged lines to make sure it's something you want to prune.

Here's an example of something you definitely want to get rid of:

```diff
--- a/vendor/dbus-0.9.7/Cargo.toml
+++ b/vendor/dbus-0.9.7/Cargo.toml
@@ -63,9 +63,5 @@
 stdfd = []
 vendored = ["libdbus-sys/vendored"]

-[target."cfg(windows)".dependencies.winapi]
-version = "0.3.0"
-features = ["winsock2"]
-
 [badges.maintenance]
 status = "actively-developed"
```

In the above example, `dbus-0.9.7` pulls in `winapi` as a dependency on Windows targets. We obviously aren't targeting Windows, so we can delete this whole chunk.

Here's another example of something you want to delete:

```diff
--- a/vendor/opener-0.7.2/Cargo.toml
+++ b/vendor/opener-0.7.2/Cargo.toml
@@ -48,7 +48,6 @@
 reveal = [
     "dep:url",
     "dep:dbus",
-    "windows-sys/Win32_System_Com",
 ]

 [target.'cfg(target_os = "linux")'.dependencies.bstr]
@@ -62,16 +61,5 @@
 version = "2"
 optional = true

-[target."cfg(windows)".dependencies.normpath]
-version = "1"
-
-[target."cfg(windows)".dependencies.windows-sys]
-version = "0.59"
-features = [
-    "Win32_Foundation",
-    "Win32_UI_Shell",
-    "Win32_UI_WindowsAndMessaging",
-]
-
 [badges.maintenance]
 status = "passively-maintained"
```

In this case, the `reveal` crate feature relies on something from `windows-sys`, so we _also_ remove that line. We know that the `reveal` feature doesn't _actually_ need `windows-sys/Win32_System_Com` on non-Windows targets, because `windows-sys` is a conditional dependency, so it's safe to remove that line.

Here's an example of something that `win-rustc-prune-list` picks up, but it's something you _don't_ want to remove:

```toml
[target.'cfg(any(unix, windows, target_os = "wasi"))'.dependencies.getrandom]
version = "0.3.0"
optional = true
default-features = false
```

`win-rustc-prune-list` just sees `windows` and flags it, so it's your job to note that it's also required for `unix`.

Finally, if you're not sure about how/if you should prune something, you can always take a look at versions of the patch from earlier Rust updates to see if a different version of the vendored crate has been pruned before. [Here's](https://git.launchpad.net/~canonical-foundations/ubuntu/+source/rustc/tree/debian/patches/prune/d-0021-vendor-remove-windows-dependencies.patch?h=merge-1.85&id=80e81b6f85ef6086177991e34100e520bd142327) one I did that has a bunch of different vendored crates.

#### A quick note on `windows-bindgen` and `windows-metadata`

You'll notice that these two crates aren't included in the exclusion list- they don't pull in `windows-sys` and friends, and Debian says they're necessary for the build process, so it's not the end of the world if they don't get pruned.

That said, previous `rustc` packages _have_ been released without either, and I've been able to get builds without `windows-metadata`, so it may be possible to exclude them fully in the future. More research is needed on this topic!

### 6. Pruning Unused Dependencies

Once you've removed all `Cargo.toml` lines which pull in unnecessary vendored dependencies, you're ready to actually update your Debian files to exclude said unnecessary dependencies from the `vendor/` directory entirely!

#### `d/prune-unused-deps`

Previous Rust maintainers have been kind enough to create a script: `debian/prune-unused-deps`. This script locates the unneeded vendored crates and adds the list to the `Files-Excluded` field of `d/copyright`. (This is that autogenerated chunk you deleted right at the start!)

In order to use the script, you need the previous Rust toolchain to bootstrap. In my opinion, it's easier to just get the toolchain using the `rustup` snap:

```shell
rustup install <X.Y.Z_old>
```

You're now ready to run the script. You just have to point it to your Rust toolchain:

```shell
RUST_BOOTSTRAP_DIR=~/.rustup/toolchains/<X.Y.Z_old>-<arch>-unknown-linux-gnu/bin/rustc \
    debian/prune-unused-deps
```

If you have issues running `d/prune-unused-deps` due to features requiring "nightly version\[s\] of Cargo", set `RUSTC_BOOTSTRAP=1` at the `cargo update` command within `d/prune-unused-deps`:

- It's possible this change can be made permanent- I just haven't got around to asking more experienced teammates if that's okay.

```diff
--- a/debian/prune-unused-deps
+++ b/debian/prune-unused-deps
@@ -24,7 +24,7 @@ done
 find vendor -name .cargo-checksum.json -execdir "$scriptdir/debian/prune-checksums" "{}" +

 for ws in $workspaces; do
-       (cd "$ws" && cargo update --offline)
+       (cd "$ws" && RUSTC_BOOTSTRAP=1 cargo update --offline)
 done

 needed_crates() {
```

#### The aftermath of `d/prune-unused-deps`

After running `d/prune-unused-deps`, you'll see a _lot_ of changes, almost none of which you actually need.

First, if you needed to edit `d/prune-unused-deps` itself earlier, make sure you restore it:

```shell
git restore debian/prune-unused-deps
```

Next, you can commit the changes to `d/control` and `d/source/lintian-overrides`. `d/prune-unused-deps` simply updated the version numbers for you.

#### Double-checking your prune work

After that, consult the new autogenerated block underneath the `Files-Excluded` field of `d/copyright`. This lists all the crates within `vendor/` that aren't needed for the source package build. Make sure that the Windows crates you worked so hard to prune earlier are included within that list! If they _aren't_ in that list, you need to go back, check for anything that pulls in those dependencies, and prune them.

Personally, I find it extra-helpful to compare my new list with the previous version's list:

```shell
git diff merge-<X.Y_old> -- debian/copyright
```

You shouldn't see anything _too surprising_. Anything removed from the diff means that that crate _used_ to be excluded, but isn't anymore. This often just means that the version number increased. You can usually see a crate with the same name and increased version number that's _new_ to the `Files-Excluded` list.

Once you've checked over your new list of excluded vendored crates, you can commit your `d/copyright` changes, restore/clean your Git repository, and continue.

### 7. Removing Vendored C Libraries

We don't _want_ to vendor Rust dependencies, we _have_ to. This is because unlike C, Rust doesn't have a stable ABI, meaning that dependencies (generally) must be statically linked to the binary.

An [excellent article](https://blogs.gentoo.org/mgorny/2021/02/19/the-modern-packagers-security-nightmare/) by a Gentoo maintainer talks more about why vendored dependencies cause problems.

Why am I talking about this? Well, we have to vendor Rust dependencies, but we _don't_ have to vendor the C libraries included within some vendored crates. This can be seen in the `Files-Excluded` field of `d/copyright`:

```
Files-Excluded:
 ...
# Embedded C libraries
 vendor/blake3-*/c
 vendor/curl-sys-*/curl
 vendor/libdbus-sys-*/vendor
 vendor/libgit2-sys-*/libgit2
 ...
```

Subdirectories _within_ vendored crates are being pruned from the orig tarball. What replaces them? The _system_ C libraries, thoughtfully provided in the Ubuntu archive. Considering the exclusion of `vendor/libgit2-sys-*/libgit2` above, let's take a look at `d/control`:

```
Build-Depends:
 ...
 libgit2-dev (>= 1.9.0~~),
 libgit2-dev (<< 1.10~~),
 ...
```

These two changes form the basis of removing vendored C dependencies.

#### Find vendored C dependencies

The easiest way I know of finding vendored C dependencies is to just search for C source files within the `vendor/` directory:

```shell
cd vendor
fdfind -e c
```

You can now check this list and figure out which of these are bundled C libraries.

#### Removing C dependencies from next orig tarball

From now on, I'll use a removal of the bundled `oniguruma` library from `rustc-1.86`, which caused some [build failures](https://pad.lv/2119556) after I failed to remove it.

Next time we run `uscan`, we want to make sure that the bundled C libraries we want to remove aren't included. To do that, simply add the C library directory to `Files-Excluded` in `debian/copyright`:

```diff
--- a/debian/copyright
+++ b/debian/copyright
@@ -50,6 +50,7 @@ Files-Excluded:
  vendor/libsqlite3-sys-*/sqlcipher
  vendor/libz-sys-*/src/zlib*
  vendor/lzma-sys*/xz-*
+ vendor/onig_sys*/oniguruma
 # Embedded binary blobs
  vendor/jsonpath_lib-*/docs
  vendor/mdbook-*/src/theme/playground_editor
```

#### Adding the system library as a build dependency

We can't just remove a C library needed by a vendored dependency without providing a proper copy of said library in its place! Instead, we can use the oniguruma Ubuntu package, `libonig-dev`. We do this by adding the package to `Build-Depends` in `d/control` AND `d/control.in`:

```diff
--- a/debian/control
+++ b/debian/control
@@ -37,6 +37,7 @@ Build-Depends:
  libgit2-dev (<< 1.10~~),
  libhttp-parser-dev,
  libsqlite3-dev,
+ libonig-dev,
 # test dependencies:
  binutils (>= 2.26) <!nocheck> | binutils-2.26 <!nocheck>,
  git <!nocheck>,
```

```diff
--- a/debian/control.in
+++ b/debian/control.in
@@ -37,6 +37,7 @@ Build-Depends:
  libgit2-dev (<< 1.10~~),
  libhttp-parser-dev,
  libsqlite3-dev,
+ libonig-dev,
 # test dependencies:
  binutils (>= 2.26) <!nocheck> | binutils-2.26 <!nocheck>,
  git <!nocheck>,
```

#### Making the vendored crate use the system library instead

In all likelihood, you'll need to adjust the vendored crate so it knows to use the system library instead of the bundled one. This can vary greatly, but it usually involves patching the crate's `Cargo.toml` or `build.rs`, so look in those places first.

In the case of `onig_sys`, I simply patched it to use the system library by default:

```diff
--- a/vendor/onig_sys-69.8.1/build.rs
+++ b/vendor/onig_sys-69.8.1/build.rs
@@ -219,7 +219,7 @@

 pub fn main() {
     let link_type = link_type_override();
-    let require_pkg_config = env_var_bool("RUSTONIG_SYSTEM_LIBONIG").unwrap_or(false);
+    let require_pkg_config = env_var_bool("RUSTONIG_SYSTEM_LIBONIG").unwrap_or(true);

     if require_pkg_config || link_type == Some(LinkType::Dynamic) {
         let mut conf = Config::new();
```

### 8. Updating Your Source Tree (Again)

#### The return of `uscan`

At this point, you've updated the list of files which must be excluded from the upstream source. This means that you need a new upstream tarball and a new Rust source.

We download the upstream Rust source code using `uscan` again, which will use your new `d/copyright` exclusion list to yank out all the stuff you just pruned:

```shell
uscan --download-version <X.Y.Z> -v 2>&1 | tee <path/to/log/output>
```

Since we've now updated our repack, we can rename the new orig tarball by changing the last number from `0` to `1`, i.e., `rustc-<X.Y>_<X.Y.Z>+dfsg0ubuntu1`, symbolizing that we've repacked the source once again.

We can also update our `debian/changelog` version number too, to something like `<X.Y.Z>+dfsg0ubuntu1-0ubuntu1`. Just make sure that the pre-hyphen bit matches the orig tarball.

#### Keeping our tree clean

In order to keep our Git tree clean, we're going to rebase all our changes on top of the new upstream Rust source.

Fair warning: I'm no Git expert. There is likely a better way to do this!

First of all, make a backup in case things go pear-shaped:

```shell
git branch backup
```

Next, we're going to do an interactive rebase, dropping the commit where we imported the upstream source:

```shell
git rebase -i merge-<X.Y_old>
```

Here's an example of what my 1.87 rebase todo looked like. Note that I'm also taking the opportunity to clean up the git tree in general, squashing my gargantuan list of vendored pruning commits into one, dropping the initial removal of the old autogenerated `Files-Excluded` chunk from `d/copyright`, etc...

```
drop f24d250953b Remove autogenerated Files-Excluded chunk
pick 222264f2705 Create 1.87.0 changelog entry
drop 93acc98c031 New upstream version 1.87.0+dfsg0ubuntu0
pick ec0e3f69232 Refresh upstream/u-ignore-ppc-hangs.patch
squash 7c73c0c7640 Refresh upstream/u-hurd-tests.patch
squash a7aecdca8cc Refresh upstream/d-ignore-test_arc_condvar_poison-ppc.patch
...
squash 8269adcb6d8 Refresh ubuntu/ubuntu-enzyme-use-system-llvm.patch
pick d045017bed6 d/p/series: Drop u-mdbook-robust-chapter-path.patch applied upstream
pick 960395225de Prune termcolor-1.1.2
squash 0c8624920e7 Prune ring-0.16.20
squash b5316157d1f Prune mio-1.0.1
...
squash d0696832bf1 Prune openssl-sys from libssh2-sys-0.2.23
pick 1239a9b4309 d/control, d/s/lintian-overrides: Update versioned identifiers
pick 713e26dc275 d/copyright: Update Files-Excluded with newly-pruned vendored crates
pick 3a590ea16cd d/copyright: Remove src/tools/rls from Files-Excluded (dropped upstream)
pick 46d9e994fbd Update changelog
```

After the interactive rebase, we're ready to go back to the previous Rust version and re-import our new orig tarball:

```shell
git checkout merge-<X.Y_old>
git checkout -b import-new-<X.Y>
```

Delete the old `experimental` branch:

```shell
git branch -D experimental
```

Then, we can merge our newly-pruned upstream source onto the previous Rust version's source cleanly:

```shell
git branch experimental
gbp import-orig \
    --no-symlink-orig \
    --no-pristine-tar \
    --upstream-branch=experimental \
    --debian-branch=import-new-<X.Y> \
    ../rustc-<X.Y>_<X.Y.Z>+dfsg0ubuntu1.orig.tar.xz
```

Finally, we can switch back to our actual branch and rebase:

```shell
git checkout merge-<X.Y>
git rebase import-new-<X.Y>
```

Consulting your `git log`, you should see the two new `gbp` commits immediately after the final commit of the last Rust version.

#### Verifying your Git changes

Before you force-push to your personal Launchpad remote, you _really_ want to make sure you didn't miss a single thing. To do that, compare your branch with your `backup` branch:

```shell
git diff backup --name-status
```

You should see ONLY a huge list of deleted `vendor/` files. If there are _any other changes_, then you made a mistake during the rebase—reset to your backup branch and try again.

Once you're 100% sure everything is good, you may delete your `import-new-<X.Y>` and `backup` branches. This is also a good time to force-push to your `<lpuser>` remote.

### 9. Refreshing the Patches (Again)

You just yanked out a ton of files, so some of the package patches will no longer apply. You must refresh all the patches so they apply cleanly onto the newly-pruned source.

- A great deal of the patches that fail to apply will do so because the vendored crates to which they apply get their version numbers updated. As usual, `git diff` is your best friend here; after moving the patch to the new version of the vendored crate, make sure nothing about the _actual content_ of the patch has changed- only the files it patches.

Hopefully, this shouldn't take too long. There will, of course, be a ton of lines removed from `d/p/prune/d-0021-vendor-remove-windows-dependencies.patch` because a bunch of the vendored crates you pruned were _themselves_ unnecessary.

### 10. Updating `XS-Vendored-Sources-Rust`

Inside of `d/control`, and `d/control.in`, there's a special field called `XS-Vendored-Sources-Rust` which must be updated. It simply lists all the vendored crate dependencies along with their versions on one _huge_ line.

Luckily, you don't have to do this manually. The `dh-cargo` package contains a script for doing that. Push all your patches, then run the script:

```shell
quilt push -a
CARGO_VENDOR_DIR=vendor/ /usr/share/cargo/bin/dh-cargo-vendored-sources
```

Copy-paste the expected value it provides to both `debian/control` AND `debian/control.in`.

- Make sure there's still an empty line after the end of the field! One time I accidentally deleted the empty line and trying to find the source of the build failure was a huge pain.

#### Outdated toolchain issues

If you're running a pre-versioned Rust Ubuntu release, then there's a decent chance the `cargo` installation required by `dh-cargo` will be too old. In this case, don't use `dh-cargo`—instead, manually download [`dh-cargo-vendored-sources`](https://git.launchpad.net/ubuntu/+source/dh-cargo/tree/dh-cargo-vendored-sources) (it's just a Perl script) and use it _without_ deb-based installations of Rust, which ensures that the Rustup snap's version will be used instead.

### 11. Updating `d/copyright`

All the new `vendor/` files must be added to `d/copyright`. Luckily, we can use a script which uses Lintian to generate all the missing copyright stanzas.

This script requires `pytoml`, so we must create and enter a virtual environment:

```shell
sudo apt install python3-venv
# Do this somewhere convenient, e.g., `~/.venvs/`
python3 -m venv rustc-lintian-to-copyright
source path/to/venv/rustc-lintian-to-copyright/bin/activate
which python3
python3 -m pip install pytoml
```

Personally, I have to manually edit `d/lintian-to-copyright.sh` in order to get it to recognize the virtual environment. This change could potentially be permanently added, but I need to check with more experienced maintainers first.

```diff
@@ -1,5 +1,5 @@
 #!/bin/sh
 # Pipe the output of lintian into this.
 sed -ne 's/.* file-without-copyright-information //p' | cut -d/ -f1-2 | sort -u | while read x; do
-       /usr/share/cargo/scripts/guess-crate-copyright "$x"
+   python3 /home/maxgmr/rustc/rustc/debian/scripts/guess-crate-copyright "$x"
 done
```

Build the source package, then run Lintian and pipe the output to the script:

```shell
dpkg-buildpackage -S -I -i -nc -d -sa
lintian -i -I -E --pedantic | debian/lintian-to-copyright.sh
```

Don't forget to leave your virtual environment!

```shell
deactivate
```

You may need to fill in some fields manually. [This](https://stackoverflow.com/questions/23611669/how-to-find-the-created-date-of-a-repository-project-on-github) is an easy way to find the start date of a GitHub repo.

Do me a favour and keep things clean by adding the new `d/copyright` stanzas alphabetically. It makes things a lot easier in the long run.

I've also created two helper scripts which make it easier to keep `d/copyright` clean. [`redundant-copyright-stanzas`](https://github.com/canonical/foundations-sandbox/blob/master/maxgmr/redundant-copyright-stanzas) produces a list of stanzas which are already covered by existing stanzas, and [`unneeded-copyright-stanzas`](https://github.com/canonical/foundations-sandbox/blob/master/maxgmr/unneeded-copyright-stanzas) produces a list of stanzas which don't apply to any files in the source tree.

- Note that both scripts are intentionally overzealous in order to catch everything- don't delete anything without manually verifying it first!

### 12. Local Build

You're now ready to try building your updated `rustc`! I use `sbuild` to do this.

First, make sure that all previous build artifacts have been cleaned from your upper-level directory to ensure that everything is built fresh:

```shell
rm -vf ../*.{debian.tar.xz,dsc,buildinfo,changes,ppa.upload}
rm -vf debian/files
rm -rf .pc
```

You also likely don't want any existing schroot sessions running:

```shell
schroot -e --all-sessions
```

Now you're ready to go!

```shell
sbuild -Ad <release> -c <schroot> .
```

#### Squashing bugs

If the build fails, it's up to you to figure out why. This will require the problem-solving skills and attention to detail I know you have!

When a particular bug gives me a lot of grief, I keep all my notes about it under a personal ["Bug Diaries" GitHub repo](https://github.com/maxgmr/bug_diaries/tree/main/rustc). I find it helpful to keep all the information I've collected in one accessible place, and perhaps it may help you address any similar bugs that appear in the future.

### 13. PPA Build

Once everything builds on your local machine, it's time to test it on all architectures by uploading it to a PPA.

#### Creating a new PPA

First, create a new PPA if this is your first PPA upload for this Rust version:

```shell
ppa create rustc-<X.Y>-merge
```

The command should return a URL leading to the PPA. You must go to that Launchpad URL and do two things:

1. "Change Details" -> Enable all "Processors"
2. "Edit PPA Dependencies" -> Set Ubuntu dependencies to "Proposed"

#### PPA changelog entry

Next, add a changelog entry, including the `~ppa` component in your version number so the PPA version isn't used in favour of the actual version in the archive:

- `<N>` is just the number of the upload. You may have to re-upload to this PPA, so you can just start with `~ppa1` for your first PPA upload, then `~ppa2` for your second, etc...

```shell
dch -bv <X.Y.Z>+dfsg0ubuntu1-0ubuntu1\~ppa<N> --distribution "<release>" "PPA upload"
```

#### Uploading the source package

Make sure that your source directory is clean (especially of `debian/files`), then build the source package:

```shell
dpkg-buildpackage -S -I -i -nc -d -sa
```

Finally, upload the newly-created source package:

- You can get the `source-changes-file` script [here](https://github.com/canonical/foundations-sandbox/blob/master/maxgmr/source-changes-file).

```shell
dput ppa:<lpname>/rustc-<X.Y>-merge $(source-changes-file)
```

### 14. Final Lintian Checks

Once everything builds in a PPA for all architectures, pat yourself on the back! The hardest part is over, and you're nearly done!

You now need to make sure that your package follows Debian policy. Clean up previous build artifacts, then build the source package once more:

```shell
dpkg-buildpackage -S -I -i -nc -d -sa
```

Now you can run Lintian with all the pedantic, experimental, etc. lints enabled. Most of the extra ones are just informational, but it's a good idea to check everything and see if there are some obvious ways to make the package just a little bit cleaner.

```shell
lintian -i -I -E --pedantic
```

Next, check the Lintian output with just the warnings and errors.

```shell
lintian -i --tag-display-limit 0 2>&1 | tee <path/to/log/file>
```

You _do_ need to address all of these. They must either be fixed or added to `debian/source/lintian-overrides{,.in}`, with a few notable exceptions:

- `E: rustc-1.86 source: field-too-long Vendored-Sources-Rust`
  - This is simply the length of the field. While we _would_ like to change this in the future in `dh-cargo`, there's nothing that can (or should) be done about this for now.
- `E: rustc-1.86 source: unknown-file-in-debian-source [debian/source/lintian-overrides.in]`
  - This is just the file used to generate the Lintian overrides for a given Rust version. It's completely harmless to have in the source tree.
- `E: rustc-1.86 source: version-substvar-for-external-package Depends ${binary:Version} cargo-<X.Y> -> rustc [debian/control:*]`
  - This is just a fallback for a non-versioned `rustc` package. While it's unlikely to ever be used, it's not a typo, so you don't need to worry about it.
- `W: rustc-1.86 source: unknown-field Vendored-Sources-Rust`
  - This is a custom field, not a typo.

Once you've dealt with all other warnings and errors, you're ready for the next step!

### 15. autopkgtests

Useful resrouces:

- [Ubuntu Maintainers' Handbook - Running Package Tests](https://github.com/canonical/ubuntu-maintainers-handbook/blob/main/PackageTests.md)
- [Ubuntu Wiki - autopkgtests](https://wiki.ubuntu.com/ProposedMigration#autopkgtests)

In order to ensure the package performs as intended, we run autopkgtests. autopkgtests are intended to test the binary package's performance. The Rust package has two autopkgtests:

1. Use the installed `rustc` package to compile the Rust compiler
2. Create a simple "Hello world" crate, build it, and run it

As you can imagine, the first autopkgtest takes considerably more time and resources. _So_ much, in fact, that you'll likely need to request more resources for the test runner.

#### Locally running the autopkgtests in the default test bed

Once you've built `rustc` in your PPA, you're ready to run the autopkgtests locally. First, create a test bed in an easily-accessible place:

```shell
autopkgtest-buildvm-ubuntu-cloud -v -r <release>
```

You'll need to create a differently-named test bed later, so I like to rename the default one:

```shell
mv autopkgtest-<release>-<arch>.img autopkgtest-<release>-<arch>-default.img
```

Now, run the autopkgtests. We're going to be simulating the [default Openstack flavour](https://wiki.ubuntu.com/ProposedMigration#autopkgtests) of 4096MiB RAM, 2 CPU cores, and 20G disk size:

- The `--log-file` option is picky. It doesn't do bash path expansions and the log file needs to exist already.

```shell
autopkgtest rustc-<X.Y> \
    --apt-upgrade \
    --shell-fail \
    --add-apt-source=ppa:<lpuser>/rustc-<X.Y>-merge \
    --log-file=<path/to/log/file> \
    -- \
    qemu \
    --ram-size=4096 \
    --cpus=2 \
    <path/to/test/bed/autopkgtest-<series>-<arch>-default.img
```

If all the autopkgtests pass, great! You're ready to [run the autopkgtests for real](#running-the-ppa-autopkgtests). However, in all likelihood, an autopkgtest fails, so you need to read on...

#### Locally running autopkgtests with more resources

If the autopkgtests failed, you likely saw something like the following near the end of the build log:

```
Did not run successfully: signal: 9 (SIGKILL)
rustc exited with signal: 9 (SIGKILL)
```

If you see this, don't panic- this is normal. The test bed has run out of RAM and killed the process. It takes a lot of resources to compile `rustc`.

All we need to do is make sure that the autopkgtests compile on the "large" Openstack test bed flavour, which has 8192MiB of RAM, 4 CPU cores, and 100G disk size.

We'll now create a second test bed which matches these conditions, and rename it accordingly:

```shell
autopkgtest-buildvm-ubuntu-cloud -s 100G --ram-size=8192 --cpus=4 -v -r <release>
mv autopkgtest-<release>-<arch>.img autopkgtest-<release>-<arch>-big.img
```

Now, try running the autopkgtests again on your new, bigger test bed:

```shell
autopkgtest rustc-<X.Y> \
    --apt-upgrade \
    --shell-fail \
    --add-apt-source=ppa:<lpuser>/rustc-<X.Y>-merge \
    --log-file=<path/to/log/file> \
    -- \
    qemu \
    --ram-size=8192 \
    --cpus=4 \
    <path/to/test/bed/autopkgtest-<series>-<arch>-big.img
```

If _these_ autopkgtests pass, we've identified the problem! The autopkgtests simply don't have enough resources.

#### Getting more resources for the autopkgtests

We need to make sure that when the autopkgtests are run for real, they're run on the larger test bed profile.

To do this, create a merge proposal in the [`autopkgtest-package-configs` repo](https://code.launchpad.net/~ubuntu-release/autopkgtest-cloud/+git/autopkgtest-package-configs) adding the new Rust version to the list of `big_packages`, which are granted the 100GB disk space, 8192MiB of memory, and 4 vCPUs we used earlier.

The change itself is trivial:

```diff
--- a/big_packages
+++ b/big_packages
@@ -174,6 +174,7 @@ rsass
 ruby-minitest
 ruby-parallel
 rustc
+rustc-<X.Y>
 rust-ahash
 rust-axum/ppc64el
 rust-cargo-c/ppc64el
```

For an example on how to format this merge proposal, you can see one of my real-life proposals [here](https://code.launchpad.net/~maxgmr/autopkgtest-cloud/+git/autopkgtest-package-configs/+merge/487781).

#### Running the PPA autopkgtests

To run the autopkgtests for real, run the following command to get links to all the autopkgtests:

```shell
ppa tests \
    --show-url ppa:<lpuser>/rustc-<X.Y>-merge \
    --release <release>
```

Click all of the links except i386 to trigger the autopkgtests for each target architecture.

You can use the same `ppa tests ...` command to check the status of the autopkgtests themselves.

The infrastructure can be a little flaky at times. If you get a "BAD" reponse (instead of a "PASS" or "FAIL"), then you just need to retry it.

### 16. Uploading the Package

You're nearly ready to request sponsorship. First, it's your duty to make your sponsor's job _as easy as possible_.

#### Sharing the right info

Go to the bug report you originally opened and create a comment with the following information:

- A link to your successfully-built PPA packages
- A link to the source code in the Foundations repository (don't forget to `git push <foundations> merge-<X.Y>`)
- A link to the commit history in the Foundations repository
- A list of any notable packaging changes
- The output of `lintian` (without any args)
- Links to all the passing autopkgtests

To see an example, here's [my 1.85 comment containing everything except the passing autopkgtest links](https://bugs.launchpad.net/ubuntu/+source/rust-defaults/+bug/2109761/comments/1). I added the passing autopkgtest links [here](https://bugs.launchpad.net/ubuntu/+source/rust-defaults/+bug/2109761/comments/3).

#### The i386 whitelist

Finally, we just need to ask an Archive Admin to add the new `rustc` package to the i386 whitelist so it can be added to the new package queue. Subscribe `ubuntu-archive` to the bug and add a comment requesting addition to the i386 whitelist.

#### Requesting review

You're now ready to get a review! Subscribe `ubuntu-sponsors` to your bug to make it visible for review and upload.

## Updating an Existing Rust Package

The `rustc` Git repository is not managed by `git-ubuntu`. This means that whenever you must fix a bug in an already-released Rust source package, you must follow the legacy `debdiff` workflow instead.

You follow the typical sequence of steps for Rust maintenance: you must update the version number with a changelog entry linking to the Launchpad bug, make your changes, build things locally, build things in a PPA, and run autopkgtests on your updated version. Make sure to update the `rustc` Git repo accordingly!

After that, however, things diverge — since the uploaded `rustc-X.Y` is not associated with the `rustc` Git repo, you're unable to generate a diff for a merge proposal.

### Creating a reviewable diff

This should be done only after verifying that the updated Rust builds in a PPA, passes all autopkgtests, and meets reasonable Lintian standards.

The Ubuntu project docs discuss the `debdiff` process in general [here](https://canonical-ubuntu-project.readthedocs-hosted.com/contributors/bug-fix/propose-changes/#submitting-the-fix).

Essentially, we need to generate a diff between the uploaded version of `rustc` and your updated version of `rustc` that doesn't rely on Git. To do this, we'll need `.dsc`s for both package versions.

Build the source package for both the new and old versions:

```shell
dpkg-buildpackage -S -I -i -nc -d -sa
```

After that, use `debdiff` to generate a diff between the two `.dsc`s. Redirect the output to an easily-accessible place:

```shell
debdiff <old_dsc> <new_dsc> > 1-<full_version_number>.debdiff
```

### debdiff patch naming convention

Let's break down an example debdiff patch name: `1-1.86.0+dfsg0ubuntu2-0ubuntu2.debdiff`

- `1-` means that this is the first revision of this patch.
- `1.86.0+dfsg0ubuntu2-0ubuntu2` is the full version number of your updated version.
  - The first `0ubuntu2` means that the orig tarball has been regenerated after the initial upload. You don't have to increment this number unless you've changed the orig tarball.
  - The second `0ubuntu2` is _always_ incremented, because you've obviously changed the package in _some_ way if you're updating it!
- The `.debdiff` suffix is simply a hint that this is a patch. Launchpad will complain (but still allow you to upload the patch) if this is not here.

Here's another example: `2-1.81.0+dfsg0ubuntu1-0ubuntu3.debdiff`

- `2-`: It's the second revision of this patch — obviously the sponsors had some feedback!
- `0ubuntu1`: The orig tarball has been unchanged since the initial upload.
- `0ubuntu3`: This package has already been updated once before since its initial upload.

### Sharing your changes for review

Instead of opening a merge proposal, you must share your patch directly underneath the bug report.

On Launchpad, click "Add attachment or patch" underneath the bug report description and add your `debdiff` as an attachment, ticking the box labelled "This attachment contains a solution (patch) for this bug". As for the comment field itself, all the [regular sponsorship request standards](#sharing-the-right-info) apply — include links to the passing autopkgtests, the PPA build, the Lintian results, and the updated Git branch itself.

After that, subscribe `ubuntu-sponsors` to the bug and await review and upload!
