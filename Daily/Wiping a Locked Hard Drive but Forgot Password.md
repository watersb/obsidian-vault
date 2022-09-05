---
created: 2022-09-05T04:17:13-06:00
updated: 2022-09-05T04:19:47-06:00
title: Wiping a Locked Hard Drive but Forgot Password
---

# Get the Disk Password

The documentation says that the backup master device password is the PSID printed on the drive label.

But this did not work for me.

My drive label shows the PSID, a 32-character code. Also on the drive are the Serial Number, Model Number, manufacturer's Part Number, and the Firmware Revision -- both text and 3of9 barcodes. I checked carefully, but a number of attempts to use the PSID as the "master" password didn't work for `hdparm`.

So perhaps I don't understand the role of the PSID, or maybe the process that set the User Password also reset the PSID to a different value.

Once more, I returned to the Google...

# `sedutil` to the Rescue 

I don't recall what led me to find this utility. I believe it was additional searching on some older terms for the drive functionality we're trying to utilize here: "Self-Encrypting Drives" (SED).

SED methods for managing passwords on the drive itself are part of a standardized implementation, a specification worked out by the Trusted Computing Group (TCG) in the early years of the 21st Century.

I don't recall at the moment -- it's been a while -- but the code name for the SED functionality was released as "Opal", level 1.0 and 2.0.

I expected `hdparm` functions to work, as it issues the SATA / SCSI commands to the hard disk according to the TCG Opal spec. But as I wasn't getting very far via `hdparm`, I looked for other CLI tools that can issue SED commands.

The error I was getting back from hdparm was always a string of hexadecimal digits, the contents of the response packet returned by the hard disk. Obviously, it was time to plop that digit string into Google and see what popped out.

A StackOverflow question hit upon this very issue: an incorrect password on a recalcitrant Seagate hard drive. Alas, the discussion there held no joy.

But another lead me to `sedutil`.

That sounds good -- a command-line tool for Linux that talks to self encrypting drives... source code on GitHub!

Ubuntu had never heard of it. That is, it wasn't in the package repository.

But the code looked simple enough, and after a bit of installation of the appropriate build tools - GNU `autoconf` and `automake` - I was able to generate the config and Makefile, build and install the tool.

```
autoreconf -f
./configure
...
make
...
make --dry-run install 
...
sudo make install
```

The tool installs in `/usr/local `as `sedutil-cli`.

Looking at the available options... Oh, look! A way to reset the drive using the PSID!

```
sudo sedutil-cli --yesIreallywanttoERASEALLmydatausingthePSID <my_psid> /dev/sdb
```

Aaand... nope. It says I have to enable the TPM - the Trusted Platform Module - on my computer, and tell the Linux SATA kernel module to load with the SATA TPM option.

# Grub

I added the option in /etc/defaults/grub -- getting lazy at this point, I just hard-coded the kernel module option on the kernel load line.

```
grub-update
```

Reboot... try the command again... a successful return value! No error!

Very exciting.

But always: **_close the loop_**. Test the results.

In this case, I rebooted the machine. Let's see what happens...

Hmm. The BIOS still shows the locked drive, and wants a password.

Dang.

# Sometimes, stupid works

I still couldn't get through with the PSID.

Look for more options...

The option shown on the man page just above the "erase all with PSID" was 

```
sedutil-cli --printDefaultPassword /dev/sdb
```

This printed out a _different_ code, the MSID.

Hmm. It's of the same form as the PSID, a 32-character alphanumeric string.

I wonder what that will do...

```
MSID="$(sedutil-cli --printDefaultPassword /dev/sdb)"

sudo hdparm --user-master m --security-unlock "${MSID}" /dev/sdb
```

**And lo! The hard drive was unlocked!**

# Make it stick

Or was it?

I wanted to get rid of this user lock once and for all, erase the drive.

```
sudo hdparm --user-master m --secure-erase-enhanced "${MSID}" /dev/sdb
```

Thanks to the "self encrypting" feature of the drive, "enhanced" was a quick operation-- it tossed the encryption key. Leaving us with terabytes of noise.

```
hdparm -I /dev/sdb
...

not enabled 
not locked
not frozen
```

This is very good!

When we read from the drive, what do we get?

```
dd if=/dev/sdb bs=4k count=64 | hexdump-C
```

Come on, feel the noise.

Mission accomplished.
