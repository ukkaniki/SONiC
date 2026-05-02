Secure Boot Overview
====================

# Table of Contents

# Purpose

This document is to give an overview of secure boot, what it does, what it doesn't do, and how enterprise network switches may make use of/implement secure boot. This does not cover any specific SONiC CLIs or SONiC implementation, and is meant to be a general guide.

# Background

Secure boot is a way to make sure only signed/verified privileged software is loaded and executed. In this context, privileged software refers to anything that might run in ring 0 (or whatever the most privileged mode is called for an architecture). This would include the bootloader and the kernel, but does not include any userspace software (including software that runs as root/with privileges).

Specifically, secure boot itself is a part of the UEFI specification, which means the terminology used here is specific to how it is used within the UEFI specification. This effectively means that when secure boot is enabled and configured, only valid, signed EFI binaries can be loaded on the system; unsigned EFI binaries, or EFI binaries signed with a X.509 certificate unknown to UEFI will be blocked. Additionally, secure boot can contain a list of EFI binaries (and their hashes) that must **not** be loaded; this could be to prevent known bad EFI binaries (i.e. having a vulnerability causing secure boot escape) from being loaded on the system, even when signed with a valid certificate.

## What's not covered by secure boot

Secure boot only says that the image being loaded (at least, the kernel being loaded) hasn't been tampered with. It doesn't say anything about any piece of hardware on the system being changed or not. For instance, the hard drive or the CPU could have changed. BIOS/UEFI settings unrelated to secure boot may have been changed as well. In addition, secure boot doesn't guarantee that, if the kernel accepts command line arguments, that hasn't changed either.

Both of the above gaps are better covered by measured boot, which, at a high level, takes a hash of various parts of the system state for later verification/use. This typically requires the hardware to have a TPM2 module (most modern CPUs have one as part of the CPU itself), and may require changes to how the OS is booted for proper/full verification.

Measured boot is orthogonal to secured boot, in that one can be enabled without the other, and both cover different things. Secure boot verifies the next thing being loaded is valid ("forward verification"), whereas measured boot makes sure that everything that has happened so far is valid/expected ("backward verification").

# Terminology

* `db`: This is the signatures database. This is the list of signatures (X.509 certificates) that are considered valid by UEFI. ELF binaries with a certificate signed with one of these signatures can be launched by the hardware.
* `dbx`: This is the forbidden signatures database. This is the list of hashes that, even if they have a valid signature as per `db`, must not be launched by the hardware.
* KEK: This is the key enrollment key database. This is the list of signatures that can be used to update the `db` or `dbx`.
* PK: This is the platform key. This key is the only key that can update the KEK. The PK also determines if secure boot is in user mode or setup mode.
* MOK: This is the machine owner key database. These are keys that are maintained by the user or machine owner that should be considered valid as well. This is only used on Linux systems that use a shim binary, to make it easier to add/remove additional custom certificates easily. This is stored as an EFI NVRAM variable on the system, and is not part of the UEFI specification.
* User mode: Mode where secure boot is enabled and a PK is present. During this time, any changes to the KEK must be signed by the PK. Clearing the PK (usually by some platform-specific method) changes secure boot to setup mode.
* Setup mode: Mode where there is no PK. During this time, the KEK can be freely changed. Setting a PK changes secure boot to user mode.

# Architecture-specific support

## amd64

Most modern hardware running on amd4 use UEFI, and most UEFI implementations likely support secure boot. The low-level protocol details appear to be [here](https://uefi.org/specs/UEFI/2.11/32_Secure_Boot_and_Driver_Signing.html).

## arm64

Hardware on arm64 may use either UEFI or uboot. If UEFI is used, secure boot may be implemented. If uboot is used, then the uboot firmware must be compiled with at least the following flags to enable a basic UEFI implementation and to enable secure boot support:

```
CONFIG_CMD_BOOTEFI=y
CONFIG_EFI_LOADER=y
CONFIG_EFI_SECURE_BOOT=y
```

Additionally, the following flags would enable using the UEFI capsule update feature, which would make it easier to update the uboot firmware:

```
CONFIG_TOOLS_MKEFICAPSULE=y
CONFIG_TOOLS_LIBCRYPTO=y
```

More details on this are available [here](https://docs.u-boot.org/en/latest/develop/uefi/uefi.html).

# Updating key databases

## Creating EFI signature lists

It is possible to update the different signature databases from within a running OS. This requires making an EFI signature list (ESL, usually having the file extension `.esl`). This is just a single file containing one or more certificates (no private keys), and is not signed/authenticated.

On Linux, this can be created with `cert-to-efi-sig-list -g $(uuidgen) secure-boot.crt secure-boot.esl`, where `secure-boot.crt` is a single X.509 certificate in PEM format, and `secure-boot.esl` is the file that will be created containing the signature list (although this list will contain, currently, just one certificate). For multiple certificates, `cert-to-efi-sig-list` will need to be executed once per certificate; the signature list files can then be normally concatenated (with `cat`).

## Signing EFI signature lists

For these EFI signature lists to be usable when the system is running under user mode, they must be signed with the keys that will be present *at the time they are loaded*. This means that if you are updating `db` or `dbx`, then they must be signed with a certificate that is in the KEK at the time the `efi-updatevar` command is run.

For each database:
* The PK ESL must be signed with the PK (in other words, with itself).
* The KEK ESL must be signed with the PK.
* The db ESL must be signed with a certificate in the KEK.
* The dbx ESL must be signed with a certificate in the KEK.

(Author's note: The author doesn't know when a signed PK ESL is actually required.)

On Linux, this can be done with `sign-efi-sig-list -k <DB1_KEY> -c <DB1_CRT> <DB2> <DB2>.esl <DB2>.auth`, where `<DB1_KEY>` is the private key of a certificate in the "authorizing" database, `<DB1_CRT>` is the public certificate in the "authorizing" database, `<DB2>` is the key database that needs to be updated (either `db`, `dbx`, `KEK`, or `PK`), `<DB2>.esl` is the unsigned ESL created in the previous step, and `<DB2>.auth` is the signed ESL that will be written.

## Updating from setup mode

If secure boot is in setup mode (meaning there is no PK set), the `db`, `dbx`, and KEK databases can be updated with just an EFI signature list (also known as an `esl` file). On Linux, this can be done with `efi-updatevar -e -f <DATABASE>.esl <DATABASE>`, where `<DATABASE>` is the key database that needs to be updated (either `db`, `dbx`, `KEK`, or `PK`), and `<DATABASE>.esl` (passed in as the value to the `-f` argument) is the ESL created in the previous step. If, instead, the certificates in the EFI signature list should be appended to instead of overwriting the database, the `-a` flag should be added to `efi-updatevar`.

Remember that the platform key is the **last** one that should be written to; otherwise, updating anything else will require signed ESLs.

## Updating from user mode

If secure boot is in user mode (meaning there is a PK set), all requests to update/overwrite `db`, `dbx`, KEK, or PK requires the ESL to be signed. With a signed ESL, on Linux, the database can be updated with `efi-updatevar -f <DATABASE>.auth <DATABASE>`, where `<DATABASE>` is the key database that needs to be updated (either `db`, `dbx`, `KEK`, or `PK`), and `<DATABASE>.auth` (passed in as the value to the `-f` argument) is the signed ESL created earlier. Note the lack of the `-e` flag. If, instead, the certificates in the EFI signature list should be appended to instead of overwriting the database, the `-a` flag should be added to `efi-updatevar`.

# Signature verification

When Secure Boot is enabled, anything that is loaded in a system-level privileged context is checked for a valid signature from a certificate stored in `db` and for the hash not being present in `dbx`. This will include (on a typical system) the bootloader, the kernel, and any kernel modules that get loaded. Any out-of-tree kernel modules will need to be manually signed with a certificate stored in `db`.

# Usage on typical consumer hardware

For simplicity and understanding's sake, it may be useful to look at how secure boot is implemented on typical consumer hardware (i.e. desktops and laptops). Some of this will likely not apply to typical enterprise setups.

The PK is a certificate typically owned by the hardware manufacturer. The KEK includes a Microsoft-owned certificate that was created specifically to be in the KEK. This certificate would then be used by Windows to update the `db`/`dbx` as necessary. The KEK may also include a certificate owned by the hardware manufacturer; this could theoretically be used by the hardware manufacturer to also update the `db`/`dbx` entries.

The `db` contains one or more Microsoft-owned certificates for booting Windows and other Microsoft-signed EFI binaries. The number of certificates and specific certificates present here depends on when the hardware was made.

## Supporting secure boot on Linux

Since there are a number of builds of the `grub` bootloader and (more recently) the `systemd-boot` bootloader, built by many Linux distros and individuals, it's not feasible to ask consumer hardware systems to have all of them signed by Microsoft. Instead, there's a small `shim` binary that is signed by Microsoft with the "Microsoft Corporation UEFI  CA 2011" certificate. This is then responsible for loading a signed bootloader (either `grub`, `systemd-boot`, or something else), as well as verifying the signature of the kernel being loaded by the bootloader. The certificates stored in both the MOK and in the UEFI db database will be considered for signature verification.

This is done by the `shim` maintaining a MOK (stored as an EFI NVRAM variable, and not modifiable from a running OS), which includes the signing signatures for the bootloader and the kernel. End users can add additional keys into the MOK (beyond what a distro may include by default).

# Possible usage in SONiC and enterprise network switches

For SONiC and for other enterprise network switches, the consumer hardware workflow doesn't make too much sense here, since a generic Windows OS isn't installed, nor is a stock Linux distro installed; it is instead some custom image, usually with a custom kernel. For that purpose, it makes more sense to have a separate workflow here, one that provides a tighter level of control.

In specific use cases, where consumers are deploying their own SONiC on OCP/SONiC-compliant networking hardware produced by OEMs/ODMs, an agreed-upon workflow to manage certificates produced by both consumers and OEMs/ODMs would be required. This is because secure boot enabled hardware runs artifacts (frimware, software, OS image etc..) owned by both entities throughout the lifetime of the product. One such workflows is discussed below.

## Approach: Staging certificate handoff

The consumer provides the OEM/ODM with a set of staging certificates — temporary credentials used solely to bootstrap the key rotation process at the consumer's facility. The OEM/ODM enrolls these staging certificates into the appropriate UEFI key databases (PK, KEK, and/or `db`) and ships the hardware with secure boot enabled.

Upon receiving the hardware, the consumer uses their staging private key to sign and apply updated ESLs that replace the staging certificates with their production certificates across the relevant databases. Once the production certificates are enrolled and validated, the consumer revokes the staging certificates by adding their hashes to `dbx`, ensuring they can no longer be used to authorize changes.

Note that between the time the hardware is shipped and the time the staging certificates are replaced, the device is only as secure as the staging credentials. Consumers should treat the staging private key with the same care as a production key, limit its validity period where possible, and ensure this rotation step is completed before the device is deployed into a production network
