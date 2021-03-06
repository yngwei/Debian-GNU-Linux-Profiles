## Deploy Heads atop coreboot for GNU/Linux
##### Copyright (c) TYA
##### Homepage: http://tya.company/

### Introduction of Heads
[Heads](https://github.com/osresearch/heads) is a boot firmware (program stored in rom to init hardware) solution based on [coreboot](https://www.coreboot.org/). [linux kernel](https://www.kernel.org/) and various tools. It has implemented OpenPGP-based signed boot, TPM-based measured boot, OTP-based attestation, and kexec.

Although "officially" supported platforms are quite limited in its main repository, Heads can actually work on at least every platform with TPM supported by coreboot.
This is because Heads consists of mainly to parts: a patched coreboot to do measurement while performing system init, and a payload consisting of a linux kernel and an initramfs for later stages. Despite Heads implements its own build system to to build both parts for its "officially" supported platforms, there is nothing prevent us to embed its payload part into our own customized coreboot image.


### Building Environment preparation for Heads
I use [this branch](https://github.com/flammit/heads/tree/coreboot-4.6) for my own deployment, since the compiler source used by master branch is too old to build easily on current Debian GNU/Linux.

Heads' build-dependency is quite similar to [those of coreboot](https://www.coreboot.org/Build_HOWTO#Requirements).

Clone it to your local machine, and then run the following command to bootstrap its environment:
```
$ make bootstrap
```
Heads' own build system will download, patch and build needed packages.

If you use newer compiler (e.g. gcc-7), several building process will fail, since newer compilers perform stricter checking. To walkaround these, you could copy [this patch file](/scripts/heads/gcc-6.3.0_ubsan-fix.patch) into unpacked coreboot source tree and [patch](/scripts/heads/lz4_fallthrough.diff) coreboot itself, then trigger bootstrapping process again:
```
$ cp /path/to/gcc-6.3.0_ubsan-fix.patch ${HEADS_ROOT}/build/coreboot-4.6/util/crossgcc/patches
$ cd ${HEADS_ROOT}
$ rm -r build/coreboot-4.6/util/crossgc/gcc-6.3.0
$ cd build/coreboot-4.6
$ patch -p1 < /path/to/lz4_fallthrough.diff
$ cd ${HEADS_ROOT}
$ make bootstrap
```

### Pre-build config for Heads
#### Disk paths
The path to boot partition and usb drive for Heads to handle are hard-written into its config file (e.g. `${HEADS_ROOT}/config/x230-generic.config`, which could be used as a template), and embedded into the initramfs, you should modify the value of `CONFIG_BOOT_DEV` and `CONFIG_USB_BOOT_DEV` to fit your own need. If it is to install a fresh-new GNU/Linux system which you want, you must determine them before you perform installation via usb boot from Heads.

#### Boot script
The default boot script `${HEADS_ROOT}/initrd/bin/generic-init` mandates interaction during boot process, which may be inconvenient for workstations and servers. If so, you could copy your own boot script into `${HEADS_ROOT}/initrd/bin/`, and modify the value of `CONFIG_BOOTSCRIPT` within the config file accordingly. [Exemplar automatic boot script](/scripts/heads/autoboot-init) and [config file](/scripts/heads/autoboot-init/x230-autoboot.config) are provided.

#### Integrated OpenPGP keyring
Heads has provided [a guide to generate key pair on OpenPGP card on the fly](https://github.com/osresearch/heads-wiki/blob/master/GPG.md), but I assume that you have already prepared an OpenPGP card with key.

Note that Heads uses the version 1 of gnupg, while current Debian GNU/Linux has made use of gnupg2, so you should also install a copy of gnupg1 to configure the gnupg keyring to be integrated:
```
$ gpg2 -o pubkey.gpg --export ${FINGERPRINT}
$ gpg2 -o seckey.stub.gpg --export-secret-keys ${FINGERPRINT}
$ gpg1 --homedir ${HEADS_ROOT}/initrd/.gnupg --import pubkey.gpg seckey.stub.gpg
$ echo "default-key ${FINGERPRINT}" > ${HEADS_ROOT}/initrd/.gnupg/gpg.conf
```

### Build Heads payload
Copy your own config file into `${HEADS_ROOT}/config`, then execute:
```
$ cd ${HEADS_ROOT}
$ make CONFIG=config/${YOUR_CONFIG} coreboot.intermediate
```
You will obtain `bzImage` and `initrd.cpio.xz` under `${HEADS_ROOT}/build/coreboot-4.6`.

### Build coreboot image
Step into your (customized) coreboot tree, copy or link `bzImage` and `initrd.cpio.xz` you just generated into it, and apply the patch provided by Heads, then configure it:
```https://www.flashrom.org/Supported_hardware
$ cd ${CBSRC}
$ patch -p1 < ${HEADS_ROOT}/patches/coreboot-4.6.patch
$ make menuconfig
```

**Note**: If your (customized) coreboot tree is based atop a revision newer than release 4.6, please make sure that [commit 35418f9814a64073550eb63a3bcb2e79021347cb](https://review.coreboot.org/19535) is reverted or never applied, otherwise PCRs (especially the PCR0~3) will be reset before Heads being executed, thus losing the measurement results of coreboot's components.

Make sure `CONFIG_MEASURED_BOOT`(`Enable TPM measured boot`) is selected, and `CONFIG_USE_OPTION_TABLE`(`Use CMOS for configuration values`, both are located within menu `General setup`) is not selected (conflict with the current version of the patch above, [a PR to fix it](https://github.com/flammit/heads/pull/3) has been filed).

Currently, the Linux kernel built from Heads only has legacy VGA text mode support, so `VGA_TEXT_FRAMEBUFFER`(located inside `Display` submenu within `Devices`) should be used.

In menu `Payload`, select `A Linux payload` for `Add a payload`, then `bzImage` for `Linux path and filename`, and `initrd.cpio.xz` for `Linux initrd`. Save your config file to `.config` and exit `manuconfig`, then `make(1)` it.
```
$ make
```

The customized coreboot image with Heads integrated will be generated at `${CBSRC}/build/coreboot.rom`. You could then flash it into your target board via [anyway possible](https://www.flashrom.org/Supported_hardware).

### Post-install config for Heads
#### Take ownership of YOUR TPM and configure it
You can only get access to the recovery shell of an unconfigured Heads, and you should configure the `tpmtotp` first:
```
# tpm-reset
```
It will ask you to type the future TPM owner password (Do not forget to back it up SECURELY) twice, after that the TPM becomes loyal to YOU.

Now seal the TPM-based totp:
```
# seal-totp
```
It will print a huge QR code on the screen. You can scan it with [FreeOTP](https://f-droid.org/en/packages/org.fedorahosted.freeotp/) to attest it later. (reboot or run ```unseal-totp```)

You need to reboot the machine to proceed.
```
# reboot
```

#### Sign the boot config files with OpenPGP card
After reboot you can get access to the boot menu. If you use the default `generic-init`, it will repeatedly ask you to make a choice while updating the totp per 30 seconds; if you chose to use `autoboot-init` provided by us, it will automatically choose 'y' after 30 seconds.

It will then mount the `/boot` partition written down in the config file, and parse the existing `grub.cfg` for you to choose, but you should always make a choice manually, until you select an option as the default one and sign it.

Choose your preferred option, and press 'd' on the next step to make it default. It will ask you "Saving a default will modify the disk. Proceed? (y/n): ", choose 'y'.

If there are encrypted volume existing on your disk, choose 'y' for "Do you wish to add a disk encryption to the TPM [y/N]: ", otherwise, choose 'n'.

Connect your OpenPGP card or token to one of the usb ports, then choose 'y' for "Please confirm that your GPG card is inserted [Y/n]: " to sign the default config. It needs the TPM owner password and the PIN for your card.

With the signature, the boot part becomes automatic, and you will have a measured (PCR 4 is used in the firmware stages.), signed, and automated boot scheme if you have chosen `autoboot-init`.

#### Update 1
[A new patch](https://github.com/persmule/heads/commit/afd3a005e078420bbbcfb8194fc90e02dcc25666) has been filed as PR with a feature not to generate hashes for initrd or modules if module.sig_enforce=1 is present, which may ease the updating of initrd from GNU/Linux OSes. The corresponding config flag has been inserted into the heads' config provided by us.

#### Update 2
With [this patch](https://github.com/persmule/heads/commit/39153d619d039665dbe4f5cad7c82ba1f72abd34), Heads could be used on platforms with no TPM available (e.g. Thinkpad x200), retaining signed boot functionality. Older platform may need [this fix](https://github.com/persmule/heads/commit/e08b7106c612696baa8f4f7e15f1e0fafd5aa678) to make use of OHCI and/or UHCI host interface to communicate with USB smart card reader.

### Reference:
[1] [Heads-wiki](https://github.com/osresearch/heads-wiki)
[2] [Heads' source code](https://github.com/osresearch/heads)
