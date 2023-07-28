---
title: "Wrecking ostree on Fedora Silverblue"
categories:
  - blog
tags:
  - linux
  - fedora
  - ostree
  - silverblue
  - immutability
---

This is the story of how I broke my Fedora Silverblue installation to the point where `rpm-ostree rollback` fails catastrophically.

<!--more-->

## Background

As anyone reading this blog should know, the `/usr` directory is a read-only partition managed by `ostree`, meaning that the user is **not** supposed to make **any** changes to it by hand. As everything is kept track by `ostree`, you can always roll back to the previous installation.

## The Problem

![XKCD - Linux security](https://imgs.xkcd.com/comics/authorization_2x.png)

The TL;DR is that I wanted to force `rpm-ostree` to require root. I don't understand why it doesn't by default considering it's writing to read-only partitions, but I digress.

Permissions in Silverblue is managed by `polkit`. The config for `rpm-ostree` is located in `/usr/share/polkit-1/rules.d/org.projectatomic.rpmostree1.rules`. I thought that if I was able to edit the file, I would be able to modify permissions for `rpm-ostree`.

Contents of the file:

```js
polkit.addRule(function (action, subject) {
  if (
    action.id == "org.projectatomic.rpmostree1.repo-refresh" &&
    subject.active == true &&
    subject.local == true
  ) {
    return polkit.Result.YES;
  }

  if (
    (action.id == "org.projectatomic.rpmostree1.install-uninstall-packages" ||
      action.id == "org.projectatomic.rpmostree1.install-local-packages" ||
      action.id == "org.projectatomic.rpmostree1.override" ||
      action.id == "org.projectatomic.rpmostree1.deploy" ||
      action.id == "org.projectatomic.rpmostree1.upgrade" ||
      action.id == "org.projectatomic.rpmostree1.rebase" ||
      action.id == "org.projectatomic.rpmostree1.rollback" ||
      action.id == "org.projectatomic.rpmostree1.bootconfig" ||
      action.id == "org.projectatomic.rpmostree1.reload-daemon" ||
      action.id == "org.projectatomic.rpmostree1.cancel" ||
      action.id == "org.projectatomic.rpmostree1.cleanup" ||
      action.id == "org.projectatomic.rpmostree1.client-management") &&
    subject.active == true &&
    subject.local == true &&
    subject.isInGroup("wheel")
  ) {
    return polkit.Result.YES;
  }
});
```

Here is the advice I got when asking in the Fedora discord server:

> `sudo mount /usr -o remount,rw` and edit the files. To restore immutability, reboot.

**DO NOT FOLLOW THIS ADVICE**

## The actual solution

`sudo cp /usr/share/polkit-1/rules.d/org.projectatomic.rpmostree1.rules /etc/polkit-1/rules.d/`

Edit `/etc/polkit-1/rules.d/org.projectatomic.rpmostree1.rules`

```js
polkit.addRule(function (action, subject) {
  if (
    action.id == "org.projectatomic.rpmostree1.repo-refresh" &&
    subject.active == true &&
    subject.local == true
  ) {
    return polkit.Result.YES;
  }

  if (
    (action.id == "org.projectatomic.rpmostree1.install-uninstall-packages" ||
      action.id == "org.projectatomic.rpmostree1.install-local-packages" ||
      action.id == "org.projectatomic.rpmostree1.override" ||
      action.id == "org.projectatomic.rpmostree1.deploy" ||
      action.id == "org.projectatomic.rpmostree1.rebase" ||
      action.id == "org.projectatomic.rpmostree1.rollback" ||
      action.id == "org.projectatomic.rpmostree1.bootconfig" ||
      action.id == "org.projectatomic.rpmostree1.reload-daemon" ||
      action.id == "org.projectatomic.rpmostree1.cancel" ||
      action.id == "org.projectatomic.rpmostree1.cleanup" ||
      action.id == "org.projectatomic.rpmostree1.client-management") &&
    subject.active == true &&
    subject.local == true &&
    subject.isInGroup("wheel")
  ) {
    return polkit.Result.AUTH_ADMIN;
  }
});
```

## Making things progressively worse

Being the idiot that I am, I followed the bad advice and edited the file directly after remounting `/usr`.

So I checked `sudo ostree fsck` at the advice of someone on `r/fedora` and it returned:

```bash
error: In commits be1231ae9dcdb3a3055ae6ae34ac0ce1b0102afbf4f9b24045cf8b4b7c6cbae1, 296473683a788a15b4a7355226f9271f083382780f655d495512bc6ec5e1063a, 25e48e9bf45cade1192a9388c0885e3afbaf529ad94daeeb3df658ecff15e20a, b07025c6212a346227dc2d8828dc320b44afa757144b785af4f747e22d9d0035:
fsck content object 5ac45fcef195a7a39cbacaa7452002b7e6299ae16f2704265770334f488b79c7:
Corrupted file object;
checksum expected='5ac45fcef195a7a39cbacaa7452002b7e6299ae16f2704265770334f488b79c7' actual='b835c9505c484ba3e8595c855c602df41c7fc1b643a8a487d9c978940f721bbb'
```

I could not find any solutions on the internet. It seems nobody else has thus far been stupid enough to do this.

At this point, `rpm-ostree` still worked.

Looking at the manual for `ostree fsck`, I found the `--delete` option... So I ran it.

This was where everything started going wrong.

```bash
$ sudo ostree fsck

Validating refs...
Validating refs in collections...
Enumerating commits...
Verifying content integrity of 382 commit objects...
fsck objects (31504/31504) [=============] 100%
3 partial commits not verified
error: 3 partial commits from fsck-detected corruption
```

Again, there is no documentation on how to delete or restore partial commits.

Trying to run `rpm-ostree rollback` returned:

```bash
Job for rpm-ostreed.service failed because the control process exited with error code.
See "systemctl status rpm-ostreed.service" and "journalctl -xeu rpm-ostreed.service" for details.
× rpm-ostreed.service - rpm-ostree System Management Daemon
     Loaded: loaded (/usr/lib/systemd/system/rpm-ostreed.service; static)
    Drop-In: /usr/lib/systemd/system/service.d
             └─10-timeout-abort.conf
     Active: failed (Result: exit-code) since Fri 2023-07-28 02:09:04 +08; 25ms ago
       Docs: man:rpm-ostree(1)
    Process: 4861 ExecStart=rpm-ostree start-daemon (code=exited, status=1/FAILURE)
   Main PID: 4861 (code=exited, status=1/FAILURE)
     Status: "error: Couldn't start daemon: Error setting up sysroot: Reading deployment 0: No such metadata object 25e48e9bf45cade1192a9388c0885e3afbaf529ad94daeeb3df658ecff15e20a.commit"
        CPU: 24ms

Jul 28 02:09:04 insignificantv5 systemd[1]: Starting rpm-ostreed.service - rpm-ostree System Management…emon...
Jul 28 02:09:04 insignificantv5 rpm-ostree[4861]: Reading config file '/etc/rpm-ostreed.conf'
Jul 28 02:09:04 insignificantv5 rpm-ostree[4861]: error: Couldn't start daemon: Error setting up sysroot…commit
Jul 28 02:09:04 insignificantv5 systemd[1]: rpm-ostreed.service: Main process exited, code=exited, stat…FAILURE
Jul 28 02:09:04 insignificantv5 systemd[1]: rpm-ostreed.service: Failed with result 'exit-code'.
Jul 28 02:09:04 insignificantv5 systemd[1]: Failed to start rpm-ostreed.service - rpm-ostree System Man…Daemon.
Hint: Some lines were ellipsized, use -l to show in full.
error: Loading sysroot: exit status: 1
```

I'm probably in the wrong but shouldn't the 2 partitions allow rollback when one of them is corrupted?

## My solution

Probably not the best one but this is how I got back to a usable state.

`sudo ostree pull fedora:fedora/38/x86_64/silverblue`

- Pull the latest version of Silverblue

`ostree log fedora:fedora/38/x86_64/silverblue`

This shows the latest commit in fedora silverblue. Copy the latest commit.

`sudo ostree deploy <latest commit>`

This will deploy the latest commit. However, the system will still be unusable. This will only fix one of the partitions.

**Note:** If this doesn't work, try pulling from `fedora:fedora/38/x86_64/testing/silverblue`. After getting a working system, you can pull `fedora:fedora/38/x86_64/silverblue` again.
{: .notice--warning}

```bash
Validating refs...
Validating refs in collections...
Enumerating commits...
Verifying content integrity of 384 commit objects...
fsck objects (132891/132891) [=============] 100%
1 partial commits not verified
error: 1 partial commits from fsck-detected corruption
```

Now there's only 1 partial commit left.

`reboot`

Make sure you boot into the partition you just deployed.

Now, run `sudo ostree deploy <latest commit>` again. This will deploy the latest commit to the other partition.

`reboot`

You now have a working system again.

**Warning** You might lose some installed packages.
{: .notice--warning}

## Conclusion

Don't mess with the filesystem.

When `rpm-ostree` breaks, `ostree` can fix it.
