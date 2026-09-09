# Fedora RISC-V workstation handoff

Date: 2026-09-09

This document is intentionally committed only on the temporary
`wip/fedora-44-riscv64-workstation-handoff` branch. Do not include it in the
upstream ParticleOS pull request.

## Overall position

The upstream-candidate branch is `fedora-44-riscv64` at commit
`5fe1cf384fa29ecb338b2756ef17b4a630271bea`, pushed to
`xjamesmorris/particleos`. It has not been submitted as a ParticleOS pull
request.

The temporary WIP branch contains an unverified follow-up for the repeating
`particleos (automatic login)` prompt:

- Preserve Fedora's `/etc/login.defs` through ParticleOS's factory `/etc`
  mechanism.
- Set Fedora RISC-V `LOGIN_TIMEOUT` to 1800 seconds during image build.
- Override the disposable RISC-V VM user's homed storage to a Btrfs subvolume,
  avoiding slow first-boot LUKS home creation under TCG.

The corrected credential path and architecture scoping pass `mkosi summary`,
but this exact combination has not completed a VM boot test yet.

## Dependency

This work depends on systemd/mkosi#4450.

Status when this handoff was written:

- State: open
- Review: approved
- Merge state: clean
- Head: `0808f37d8c9d74c337b341f6b519279b74a0bccf`

The ParticleOS `MinimumVersion=commit:` pin matches that head. Recheck it before
building because a rebase or squash merge will require updating the pin.

## Workstation setup

```sh
gh repo clone xjamesmorris/particleos
cd particleos
git switch --track origin/wip/fedora-44-riscv64-workstation-handoff

gh repo clone systemd/mkosi ../mkosi
git -C ../mkosi fetch origin pull/4450/head:fedora-44-riscv64-vm
git -C ../mkosi switch fedora-44-riscv64-vm
test "$(git -C ../mkosi rev-parse HEAD)" = \
    0808f37d8c9d74c337b341f6b519279b74a0bccf

cp mkosi.local.conf.workstation.example mkosi.local.conf
```

Secure Boot/PCR keys are deliberately not committed. Either transfer
`mkosi.key` and `mkosi.crt` securely outside GitHub or generate workstation
test keys:

```sh
../mkosi/bin/mkosi genkey
```

## Exact local build configuration

`mkosi.local.conf.workstation.example` reproduces the laptop test variant:

- Fedora 44
- `riscv64`
- `fastfetch`
- `sudo`

The file is only a transfer aid. Copy it to the ignored `mkosi.local.conf`
before building.

## Current source changes

- `mkosi.conf.d/fedora/mkosi.conf.d/riscv64.conf`
  - Loads the RISC-V-only credential override directory.
- `mkosi.conf.d/fedora/riscv64.credentials/home.create.particleos`
  - Reuses the normal VM user record but sets `"storage": "subvolume"`.
- `mkosi.postinst.chroot`
  - For Fedora RISC-V only, changes `LOGIN_TIMEOUT` in `/etc/login.defs` to
    1800 seconds before the factory `/etc` snapshot is made.
- `mkosi.extra/usr/lib/tmpfiles.d/etc.conf`
  - Adds the previously missing `/etc/login.defs` factory link.

## Established results

- Fedora RISC-V resolves to Fedora 44; Fedora x86-64 remains rawhide.
- The full signed/verity image builds.
- QEMU discovers `/dev/tpm0`.
- `systemd-repart` creates the TPM-encrypted root and swap.
- Root unlock, root mount, network-online, and `multi-user.target` have all
  completed successfully.
- The built manifest includes `fastfetch` and `sudo`.
- `../mkosi` has no working-tree changes.

## Autologin investigation

The original VM credential creates a password-bearing homed user without an
explicit storage type. On an unencrypted `/home`, homed defaults to a LUKS
home. Under RISC-V TCG:

1. `systemd-homed-firstboot` reported:
   `Operation on home particleos failed: Connection timed out`.
2. `agetty` repeatedly started `particleos (automatic login)`.
3. util-linux `login` used its compiled 60-second timeout because ParticleOS
   did not link the packaged `/etc/login.defs` from the factory.

Experiments showed:

- Setting only a subvolume home did not fix the loop while `login` still used
  its 60-second compiled default.
- Setting only a longer factory `LOGIN_TIMEOUT` did not fix slow LUKS home
  creation.
- Root autologin eventually reached a shell but still looped through several
  60-second login attempts before the factory link was fixed.
- The current WIP combines the factory link, longer login timeout, and
  RISC-V-only subvolume home. This exact combination remains to be tested.

## First required action

Build and boot the exact WIP state:

```sh
../mkosi/bin/mkosi -B -f
../mkosi/bin/mkosi vm
```

Success criteria:

- No repeated `particleos (automatic login)` banners.
- No `Password:` prompt.
- No `login: timed out` message.
- A direct `particleos@...$` shell appears.
- `systemctl is-system-running` and `systemctl --failed --no-pager` can be
  inspected from that shell.

In tmux with `Ctrl+A` as the tmux prefix, exit QEMU with:

```text
Ctrl+A, Ctrl+A, X
```

## Re-evaluate timeouts on the workstation

The laptop was heavily loaded and RISC-V was running under TCG. The workstation
may not need the conservative timing extensions.

Use the current values for the first reproducible boot, then measure and reduce
them:

- `systemd.default_device_timeout_sec=1800` is VM-only and should be reduced to
  900, 600, 300, or removed if reliable.
- `systemd.default_timeout_start_sec=300` may also be reducible.
- The RISC-V `LOGIN_TIMEOUT 1800` override may be reducible or removable if the
  subvolume-backed VM user makes login reliably complete within Fedora's
  default 60 seconds.

Do not retain large timeouts merely because the laptop needed them. Record
monotonic boot timings and choose the smallest values that reliably pass
repeated cold first boots.

## After validation

1. Review whether the generic `/etc/login.defs` factory link should be a
   separate ParticleOS fix.
2. Keep only the minimum RISC-V-specific workaround required on the
   workstation.
3. Amend or create focused commits.
4. Update `WIP-PR-DESCRIPTION.md`.
5. Move the validated commits onto `fedora-44-riscv64`.
6. Push with `--force-with-lease` only if the existing fork branch must be
   rewritten.

## Non-transferred local artifacts

Ignored image outputs, signing keys, and detailed session logs are not pushed
to GitHub. Important conclusions are captured above. Rebuild images on the
workstation rather than copying the laptop's large `mkosi.output` directory.
