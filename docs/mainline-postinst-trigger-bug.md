# Mainline 7.1.x packaging bug: /etc/kernel/postinst.d silently skipped

Analysis dated 2026-07-08, on Ubuntu with `mainline` 1.4.14 installing kernel
`7.1.3-070103-generic` from the Ubuntu mainline PPA
(`linux-image-unsigned-7.1.3-070103-generic_7.1.3-070103.202607041245_amd64.deb`).

## Symptom

After `sudo mainline install-latest` completes without any error:

```console
$ sbverify --list /boot/vmlinuz-7.1.3-070103-generic
No signature table present
$ ls -l /boot/initrd.img
lrwxrwxrwx 1 root root 31 ... /boot/initrd.img -> initrd.img-7.1.3-070103-generic   # dangling!
$ ls /boot/initrd.img-7.1.3-070103-generic
ls: cannot access ...: No such file or directory
```

The kernel is unsigned **and** has no initramfs, and grub was never updated.
It is not just this repository's signing hooks that were skipped — *nothing*
in `/etc/kernel/postinst.d` ran (dracut/initramfs-tools, `zz-update-grub`,
`zz-shim`, `zz-mainline-signing`, ...). The new kernel is unbootable.

`/var/log/dpkg.log` gives it away: the whole trigger processing for the
kernel package completes within a single second, which is impossible if an
initramfs had been generated.

## Root cause

Ubuntu kernel packages do not run `/etc/kernel/postinst.d` directly from the
postinst. They write a small script to `/usr/lib/linux/triggers/$version` and
fire a dpkg trigger (`linux-update-$version`), so that hooks run only once
even when several related kernel packages are configured in one dpkg run.
When the trigger is processed, the postinst (called with `$1 = triggered`)
executes that script and deletes it.

**Correct packaging** (7.0.x mainline, Ubuntu archive kernels) writes one
trigger script covering both hook directories in a single `run-parts` call,
`/etc` first:

```sh
cat - >/usr/lib/linux/triggers/$version <<EOF
DEB_MAINT_PARAMS="$*" run-parts --report --exit-on-error --arg=$version \
    --arg=$image_path /etc/kernel/postinst.d /usr/share/kernel/postinst.d
EOF
dpkg-trigger --no-await linux-update-$version
```

**Broken packaging** (7.1.x mainline) loops over the directories and writes
the trigger script once per directory — with `>`, which truncates:

```sh
for d in /etc/kernel/postinst.d /usr/share/kernel/postinst.d; do
    if [ ! -d $d ]; then
        continue
    fi
    mkdir -p /usr/lib/linux/triggers
    cat - >/usr/lib/linux/triggers/$version <<EOF
DEB_MAINT_PARAMS="$*" run-parts --report --exit-on-error --arg=$version \
    --arg=$image_path $d
EOF
    dpkg-trigger --no-await linux-update-$version
done
```

The second iteration overwrites the first, and the two `dpkg-trigger` calls
for the same trigger name collapse into one activation. When the trigger is
processed, only the *last existing* directory runs.

`/usr/share/kernel/postinst.d` is shipped by `linux-base` (it contains
`touch-reboot-required`), so on any current Ubuntu system it exists — and
`/etc/kernel/postinst.d` is therefore always the one that gets dropped. The
only observable effect of the trigger is a `touch /run/reboot-required`,
which is why it finishes instantly and why nothing fails: every hook is
skipped *successfully*.

The fix upstream is a one-liner: `>` → `>>` (or restoring the single
`run-parts` invocation listing both directories).

## Diagnosis checklist

To confirm a system/kernel is affected:

1. `grep -A12 'for d in' /var/lib/dpkg/info/linux-image-*VERSION*.postinst`
   — shows the per-directory loop with the truncating `cat - >` redirect.
2. `/boot/initrd.img-VERSION` missing even though the package configured
   without error.
3. All dpkg.log entries for the kernel's `trigproc` fall within ~1 second.
4. `sbverify --list /boot/vmlinuz-VERSION` → `No signature table present`.

## Workaround shipped in this repository

A pair of hooks (see the [README section](../README.md#workaround-mainline-71x-kernels-skip-etckernelpostinstd)
for install commands):

- [`sbin/00-etc-kernel-hooks-stamp`](../sbin/00-etc-kernel-hooks-stamp),
  installed in `/etc/kernel/postinst.d`, records a per-version stamp in
  `/run/etc-kernel-postinst-ran/` whenever that directory runs.
- [`sbin/zz-run-etc-kernel-hooks`](../sbin/zz-run-etc-kernel-hooks),
  installed in `/usr/share/kernel/postinst.d` (which always runs). Stamp
  present → consume it and do nothing (correct packaging). Stamp missing →
  `/etc/kernel/postinst.d` was skipped, so run it via `run-parts` with the
  same version/image arguments.

Properties worth preserving if these scripts are modified:

- The stamp is **consumed on every path**, including failure, so it can never
  go stale and suppress a later run of the same version in the same boot
  (`/run` is tmpfs, so reboots clear it anyway).
- A failing hook's exit code is propagated so dpkg still reports the error
  (`run-parts --exit-on-error` semantics are kept).
- `DEB_MAINT_PARAMS` is inherited from the trigger environment, so hooks see
  the same value as in the non-buggy path.
- No double-run on correct packaging: `run-parts` executes `/etc` before
  `/usr/share`, so the stamp always exists by the time the shim runs.

## Recovering a kernel installed while affected

Run the skipped directory once by hand (signs the kernel, builds the
initramfs, updates grub):

```bash
sudo run-parts --report --exit-on-error \
  --arg=VERSION-generic --arg=/boot/vmlinuz-VERSION-generic \
  /etc/kernel/postinst.d
```

## Upstream status

Not yet reported as of 2026-07-08. Should be filed on Launchpad against the
Ubuntu mainline kernel packaging (the `debian/templates` image postinst used
by kernel-ppa mainline builds). The workaround hooks can be retired once
fixed packages ship — the shim is inert on correct packaging either way.
