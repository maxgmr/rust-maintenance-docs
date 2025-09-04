# Rust Version Strings

As mentioned elsewhere, the Rustc version string format used on our archives is rather complicated.
It's a layer-cake of Debian convention, our convention, and legacy code.
Although you will usually change only a few parts of it, it's a good idea to know what all of it means.

## The Full Breakdown

Here, `{curly_braces}` indicate placeholders to be edited, and `[square braces]` indicate optional parts.

```
{rustc_version}+dfsg0ubuntu{repack}[~bpo{vendored_deps}]-0ubuntu{revision}.[{ubuntu-release}][~ppa{PPA}]
```

> [!NOTE]
> There are a few different version string schemas floating around the archives.
> This document is about how future version strings SHOULD be made, not how they used to be made.
> This is also why I can't give links these versions; some of them are not real!
> They're made up for example purposes.

<!-- TODO: I believe my (ppark's) rustc 1.83->noble port is the first backport that properly uses
this spec. However I messed up part of the version string and misreport the vendored deps.
(I don't vendor libgit2 but the version string claims I do.)

I will update this page once a backport *without* versioning mistakes is made -->

## `{rustc_version}`

> *The rustc version.*

This is simple; it's just the version of rustc.

Examples:
- `1.88.0+dfsg0ubuntu1-0ubuntu1~ppa1`: rustc 1.88.0
- `1.87.0+dfsg0ubuntu1-0ubuntu1`: rustc 1.87.0
- `1.85.1+dfsg0ubuntu2-0ubuntu2~ppa3`: rustc 1.85.1
- `1.80.0+dfsg0ubuntu1~bpo2-0ubuntu0.24.09~ppa4`: rustc 1.80.0

## `+dfsg0ubuntu{repack}`

> *The number of times you have edited Files-Excluded.*

`dfsg` is short for "Debian free software guidelines."
The presence of `+dfsg{whatever}` in any package indicates that the orig tarball has been *repacked* in some way.

Usually, this is done for copyright reasons.<sup id="anchor1">[1](#footnote1)</sup>
However, in our case, we are doing it just to make our tarballs smaller.
Rustc comes with lots of functionality that we don't need on our archives, most notably Windows support.
To save space, we (ab)use Debian's ability to *exclude* all those unnecessary files.
That way, we aren't hauling around megabytes of code the compiler is going to ignore anyways.

So, this part of the version string indicates that Debian (`dfsg`) has repacked the tarball `0` times,
and that we (`ubuntu`) have repacked it `{repack}` times.

The files to be excluded are in `debian/copyright`, in the `Files-Excluded:` field.
If you have edited that field since the last release (and thus changed the contents of the orig tarball), increment `{repack}` by one.
**It starts at 1.**
Generally, you will only have to change the excluded files in backports -- this is because the most common reason to repack is if you change if `libgit2` or LLVM are vendored.
Although we do vendor *crate* dependencies for normal ports, this number only cares about when we *change* those vendors, and 99% of the time the upstream vendored crates are just fine.

Examples:
- `1.88.0+dfsg0ubuntu1-0ubuntu1~ppa1`: 1st repack
- `1.87.0+dfsg0ubuntu1-0ubuntu1`: 1st repack
- `1.85.1+dfsg0ubuntu2-0ubuntu2~ppa3`: 2nd repack
- `1.80.0+dfsg0ubuntu1~bpo2-0ubuntu0.24.09~ppa4`: 1st repack

## `[~bpo{vendored_deps}]`

> *Code number for what dependencies are vendored.*

When you are backporting, you sometimes have to vendor dependencies.
(See [`backporting-rustc.md`](./backporting-rustc.md).)

If you have vendored any dependencies, then you need to insert this part.
If you have not vendored any dependencies, then you do not include *anything* here.

`{vendored_deps}` is a code number:
- `0` when `libgit2` *and* LLVM are vendored
- `2` when *ONLY* LLVM is vendored, and `libgit2` is from the archive.
- `10` when *ONLY* `libgit2` is vendored, and LLVM is from the archive.

Examples:
- `1.88.0+dfsg0ubuntu1-0ubuntu1~ppa1`: No vendored deps
- `1.87.0+dfsg0ubuntu1-0ubuntu1`: No vendored deps
- `1.85.1+dfsg0ubuntu2-0ubuntu2~ppa3`: No vendored deps
- `1.80.0+dfsg0ubuntu1~bpo2-0ubuntu0.24.09~ppa4`: Only LLVM is vendored
- `1.83.0+dfsg0ubuntu1~bpo0-0ubuntu0.24.09~ppa1`: Both `libgit2` and LLVM are vendored

## `0ubuntu{revision}`

> *The "real" version.*

This is the component that actually indicates the number of times you have edited this particular port.
Every time you need to edit a particular version of Rustc on a particular version of Ubuntu, you increment this number.
**It starts at 0,** but any versions you push to an archive should start at 1.
0 indicates the theoretical "pristine" version before you make any changes.
(TODO this is *probably* correct but there is some lore here we need to get from Zixing before finalizing this section)

For example, I (ppark) have used this when I accidentally published a version
`1.83.0+dfsg0ubuntu1~bpo0-0ubuntu0.24.03` that I thought was ready to merge, but had some lingering lintian errors.
Thus, I fixed the lintian errors (in a few rounds) and eventually published version
`1.83.0+dfsg0ubuntu1~bpo0-0ubuntu1.24.03`.<sup id="anchor2">[2](#footnote2)</sup>
[You can see that whole saga here.](https://launchpad.net/~petrakat/+archive/ubuntu/rustc-1.83-merge/+packages?field.name_filter=&field.status_filter=&field.series_filter=)


Examples:
- `1.88.0+dfsg0ubuntu1-0ubuntu1~ppa1`: First Revision
- `1.87.0+dfsg0ubuntu1-0ubuntu1`: First Revision
- `1.85.1+dfsg0ubuntu2-0ubuntu2~ppa3`: Second Revision (#2)

## `{ubuntu-release}`

> *The Ubuntu release in case of backport, plus some hacks.*

In theory, this part is simple: it's the number for the Ubuntu release this backport is for.
In practice, however, there's a catch; while the backport is a work-in-progress, decrement the last number by 1.
That way, it always sorts before the finalized version. This is a little bit of a hack, but it works.

It is omitted entirely if this is not a backport.

Examples:
- `1.88.0+dfsg0ubuntu1-0ubuntu1~ppa1`: Not a backport
- `1.87.0+dfsg0ubuntu1-0ubuntu1`: Not a backport
- `1.80.0+dfsg0ubuntu1~bpo2-0ubuntu0.24.09`: WIP, for Ubuntu 24.10 (that's Oracular Oriole)
- `1.83.0+dfsg0ubuntu1~bpo0-0ubuntu1.24.03~ppa3`: WIP, for Ubuntu 24.04 (that's Noble Numbat)
- `1.83.0+dfsg0ubuntu1~bpo0-0ubuntu1.24.04`: Complete, for Ubuntu 24.04 (again, Noble Numbat)

## `~ppa{PPA}`

> *The number of times you have pushed it to your PPA for testing.*

Every time you make some changes, and want to check that it builds and passes tests by pushing it to your PPA,
you should increment this number.
This is because Launchpad does not let you "re-upload" a version with the same version string but different source code.
Thus you have to make each PPA upload's version string be different, and this is our convention for doing that.

Every time you change the rest of the version string in some way, you can reset this to 1.

If this part is *not* present, that means it's on the main archive, so it's a version that's actually out!

> [!NOTE]
> Versions with `~ppa{PPA}` parts should never make it onto a VCS system; they are only for the benefit of the PPA.
> The workflow is usually to add a new changelog entry with a `~ppa` version that just says `Upload to PPA`,
> use `dput` to upload it, then revert to the old changelog.

Examples:
- `1.88.0+dfsg0ubuntu1-0ubuntu1~ppa1`: First push to your PPA
- `1.87.0+dfsg0ubuntu1-0ubuntu1`: No PPA (i.e., real complete release)
- `1.85.1+dfsg0ubuntu2-0ubuntu2~ppa3`: 3rd push to your PPA

## Putting it All Together

Let's do a complete breakdown on some example version strings.

| String | `{rustc_version}` | `{repack}` | `~bpo{vendored_deps}` | `{revision}` | `{ubuntu-release}` | `~ppa{PPA}` |
| ------ | ------------- | ------ | ------------- | -------- | -------------- | --- |
| `1.80.0+dfsg0ubuntu1~bpo2-0ubuntu0.24.09~ppa4` | 1.80.0 | #1 | 2, so only LLVM | 0 | In-dev backport for 24.10 (OO) | 4 |
| `1.88.0+dfsg0ubuntu1-0ubuntu1~ppa1` | 1.88.0 | #1 | No vendored deps | 1 | Omitted b/c this is a normal port | 1 |
| `1.83.0+dfsg0ubuntu1~bpo0-0ubuntu1.24.03~ppa3` | 1.83.0 | #2 | 0, so both libgit2 and LLVM | 1 | In-dev backport for 24.03 (NN) | 3 |

---

<b id="footnote1">Note 1:</b>
Say that a package, `libfoo`, has a few files in it that are covered by non-free licenses.
Therefore we need to exclude those files before we send it to the archive, because otherwise we would be redistributing code illegally.

However, in order to do this with a *patch*, we would need to put all the copyrighted code into the patch!
We would still be illegally redistributing code.

To get around this, we use the `Files-Excluded:` field to omit files by *name*.
In our case (as said above), we're doing this for convenience, not for legal reasons.
It's perfectly legal to distribute all the Windows interop code, it's just a waste of space. [↩](#anchor1)

<b id="footnote2">Note 2:</b>
If you go looking for this version on the archive, you should know that I messed up the `~bpo{vendored_deps}` part.
I have `~bpo0`, but it should be `~bpo2` because I only vendored LLVM.
It was my first backport ... [↩](#anchor2)
