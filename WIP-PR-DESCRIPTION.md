> [!IMPORTANT]
> This draft belongs to the temporary workstation-handoff branch. Validate the
> autologin WIP and minimize the laptop-derived timeouts before submission.

## Summary

Add a Fedora 44 `riscv64` configuration for building and booting the
base/headless ParticleOS image.

- Select Fedora 44 for `riscv64`; Fedora rawhide currently does not provide the
  required RISC-V repository content.
- Extend timeouts for emulated RISC-V guests.
- Configure QEMU to expose its software TPM through the device tree.
- Disable expected-PCR signing on RISC-V because Fedora's RISC-V EDK2 firmware
  does not initialize measured-boot PCRs.

## Dependency

This PR depends on systemd/mkosi#4450, which adds Fedora RISC-V repository,
firmware, and VM support.

The configuration currently requires that PR's approved head,
`0808f37d8c9d74c337b341f6b519279b74a0bccf`, through `MinimumVersion=commit:`.
The pin must be updated if the mkosi PR is rebased or squash-merged.

## TPM and measured boot

On QEMU RISC-V, `tpm-tis-device` is described through the device tree rather
than ACPI. Fedora's RISC-V EDK2 firmware currently selects ACPI and does not
initialize the PCRs needed by ParticleOS's expected-PCR policy.

The RISC-V drop-in therefore:

- explicitly enables mkosi's software TPM backend;
- attaches `tpm-tis-device`;
- disables ACPI for the VM so Linux discovers the TPM through the device tree;
- disables expected-PCR signing for this architecture.

TPM-backed encryption of root and swap remains enabled. Secure Boot signing,
the signed UKI, and the dm-verity-protected `/usr` remain enabled. The
architecture-specific loss is measured-boot PCR policy enforcement until the
RISC-V firmware supports it.

## Validation

Validated against systemd/mkosi#4450 at
`0808f37d8c9d74c337b341f6b519279b74a0bccf`:

- Fedora `riscv64` resolves to Fedora 44 and a disk image.
- Fedora `x86-64` remains on rawhide.
- `mkosi -B -f` builds the complete signed and verity-protected image.
- Normal `mkosi vm` boots have discovered `/dev/tpm0`, completed
  `systemd-repart`, unlocked TPM-encrypted root and swap, mounted root and
  verity-protected `/usr`, and reached network-online and `multi-user.target`.
- A customized build manifest includes `fastfetch` and `sudo`.

The remaining WIP is to validate that the RISC-V-only subvolume-backed VM user
and persistent login configuration eliminate the repeated autologin prompt.

## AI assistance

GitHub Copilot was used to develop, debug, review, and validate this change.
