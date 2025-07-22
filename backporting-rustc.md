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

For example, if you're backporting `rustc-1.82` to Jammy...

- `<X.Y>` = 1.82
- `<X.Y.Z>` = 1.82.0
- `<X.Y_old>` = 1.81
- `<X.Y.Z_old>` = 1.81.0
- `<release>` = Jammy
- `<source_release>` = Noble

`<lpuser>` refers to your Launchpad username.
