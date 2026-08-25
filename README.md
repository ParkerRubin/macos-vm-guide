# macos-vm-guide
Step-by-step guide to running macOS Ventura in a VMware VM on Windows (Intel/AMD), using the OpenCore Boot VMDK method. No .vmx editing required.

# Setting Up a macOS VM on VMware (Intel & AMD)

Hi, after mass googling to 1:00 am and downloading 16GB files at 2-25 MB/s, you now have the blueprint to set this up. Lucky you.

> **Heads up:** running macOS on non-Apple hardware technically goes against Apple's software license terms. Nobody's coming after you for a personal VM, but worth knowing it's not "sanctioned."

## Table of Contents

- [What you'll end up with](#what-youll-end-up-with)
- [1. Get the Ventura ISO](#1-get-the-ventura-iso)
- [2. Install VMware Workstation Pro](#2-install-vmware-workstation-pro)
- [3. Apply the Unlocker](#3-apply-the-unlocker)
- [4. Create the VM](#4-create-the-vm)
- [5. The part that actually trips people up: OpenCore](#5-the-part-that-actually-trips-people-up-opencore)
- [6. Verify the ISO before you boot](#6-verify-the-iso-before-you-boot)
- [7. Install macOS](#7-install-macos)
- [8. VMware Tools](#8-vmware-tools)
- [9. Install Xcode](#9-install-xcode)
- [Troubleshooting](#troubleshooting)

## What you'll end up with

A working macOS Ventura VM on Windows, running through VMware Workstation Pro, using an OpenCore Boot VMDK so you never have to hand-edit a `.vmx` file. Built for coursework that needs Xcode and doesn't want to fight a laggy RDP Mac every week.

## 1. Get the Ventura ISO

The first step is the longest, get it over with.

- **Source:** [archive.org/download/macos_iso](https://archive.org/download/macos_iso)
- Grab `Ventura_13.0.1.iso` and `Ventura_13.0.1.iso.sha256sum`. Skip everything else in that folder (the other macOS versions, the `.dmg` files).

You can pull this two ways:

| Method | Speed (in my experience) |
|---|---|
| Direct HTTP download from the archive.org page | Slower, but reliable |
| Torrent (`macos_iso_archive.torrent`, same page) via [qBittorrent](https://www.qbittorrent.org/) | Faster when it works, but see the warning below |

If you go the torrent route, open qBittorrent, add the `.torrent` file, then **uncheck every file except the two listed above** before starting the download, or you'll pull the entire 77GB collection.

> **Real talk on the torrent:** it can fail silently. Mine reported the file as fully downloaded at the right size, and it still failed the checksum, twice in a row. If qBittorrent closes on its own mid-download, don't trust the result even if the file looks complete. If you keep hitting checksum mismatches, switch to the direct HTTP link on the same archive.org page instead. It's slower, but it finished clean on the first try when the torrent kept failing me.

## 2. Install VMware Workstation Pro

Now free for public/personal use.

- [VMware Workstation Pro download](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion)

**Make sure it's the Pro version.**

## 3. Apply the Unlocker

The "Apple macOS" guest OS option only shows up in VMware when it thinks it's running on Apple hardware. This patches that.

- [auto-unlocker (GitHub)](https://github.com/paolo-projects/auto-unlocker)

Close VMware completely, run the unlocker executable, then reopen VMware. "Apple macOS" should now appear as a guest OS option when creating a new VM.

## 4. Create the VM

Not going to walk through every wizard screen here, either watch a video or ask an AI assistant while you click through it, since it involves a lot of niche settings: SATA config, resource allocation, CPU cores.

**Quick things to get right:**

- Keep the virtual disk type on **SATA**, not NVMe, or macOS won't boot off it.
- Whatever CPU allocation you set in the VM's Processors tab, make sure it's a number your actual machine can support. A mismatch here is a common reason the VM won't power on cleanly.
- Guest OS: **Apple macOS**, Version: **macOS 13**.
- Give it at least 8GB of RAM and 80GB of disk.

## 5. The part that actually trips people up: OpenCore

The install method here uses an **OpenCore Boot VMDK** as a second hard disk. This avoids editing the `.vmx` file by hand, but it means your VM has two hard disks instead of one, and VMware needs to know which one to boot from first.

1. Create the VM as normal and let VMware make its own blank virtual disk. This will be your macOS install target.
2. Download an OpenCore Boot VMDK that matches your CPU (Intel or AMD) and a core count option (commonly 4, 8, or 16). Extract it, it's a single `.vmdk` file.
3. In VM Settings, select your existing hard disk, click **Advanced**, and change the **Virtual Device Node** from `SATA 0:0` to `SATA 2:0`.
4. Click **Add** → **Hard Disk** → **SATA** → **Use an existing virtual disk**, and browse to the OpenCore `.vmdk`. When prompted about format, choose **Keep Existing Format**, not Convert.
5. Select the new OpenCore disk, click **Advanced**, and set its Virtual Device Node to `SATA 0:0`. This makes OpenCore the first thing the VM boots.
6. The CD/DVD drive holding your Ventura ISO can sit on any node that isn't already taken. `SATA 1:0` is fine and is usually the default.

> **The core count confusion:** the OpenCore VMDK core count does not need to match your physical CPU. It needs to match the number of cores you allocate to the VM in Processors settings. If you give the VM 8 cores, download the 8-core VMDK, regardless of whether your actual CPU has 6, 8, or 16 cores. Getting this wrong is a common reason the VM boot loops or never gets past a black screen.

## 6. Verify the ISO before you boot

Don't skip this. Open PowerShell as admin in the folder with your ISO:

```powershell
Get-FileHash Ventura_13.0.1.iso -Algorithm SHA256
```

Compare the output against the contents of `Ventura_13.0.1.iso.sha256sum`. If they match, you're clear. If they don't, redownload the ISO before doing anything else, it will not install correctly otherwise, and the failure mode (a silent black screen) is hard to diagnose after the fact.

## 7. Install macOS

Power on the VM. You'll land on the OpenCore boot picker.

- **What to expect on the boot screen:** an Apple logo with a progress bar means it's working, let it ride. A plain black screen with just a blinking cursor for more than about 5 minutes means it's actually stuck. Power off and double-check your VM settings before trying again.

From there:

1. Select **Install macOS Ventura** from the OpenCore menu.
2. Wait for the Recovery environment to load (can take several minutes and may look frozen, be patient).
3. Open **Disk Utility**, select the VMware Virtual SATA Hard Drive (your 80GB disk), click **Erase**. Name it `Macintosh HD`, format `APFS`, scheme `GUID Partition Map`.
4. Close Disk Utility, select **Install macOS Ventura** again, agree to the terms, and pick `Macintosh HD` as the install target.
5. It'll copy files and reboot several times over roughly 20-30 minutes. Don't interfere, let it auto-select the installer from the OpenCore menu each time it restarts.

You don't need to sign in with an Apple ID at any point in this process, but do as you wish.

## 8. VMware Tools

Worth installing, gives you proper screen resolution, shared clipboard, and drag-and-drop file transfer between host and guest.

- VM menu → **Install VMware Tools**.
- You'll likely get a "could not find component" error. It gives you a direct link to `darwin.iso`, download that.
- Mount `darwin.iso` as the VM's CD/DVD image, restart the VM.
- Run the VMware Tools installer from the desktop icon. If macOS blocks it under **Privacy & Security → Security**, click **Allow**, enter your password, and re-run the installer.
- Restart once more.

## 9. Install Xcode

- **Do not use the App Store.** A fresh Ventura install will only offer you the newest Xcode, which now requires macOS 26 and Apple silicon. It will not install here.
- Xcode 14.3.1 is the version actually built for Ventura. Grab it directly:

  [Xcode 14.3.1 direct download](https://download.developer.apple.com/Developer_Tools/Xcode_14.3.1/Xcode_14.3.1.xip)

- You'll need to sign in with an Apple ID on developer.apple.com to download it. A free Apple ID is enough, no paid developer account required.
- Double-click the `.xip` to expand it, drag the resulting `Xcode.app` into Applications, launch it, and click through the first-run component install.

## Troubleshooting

**Black screen after selecting Install macOS Ventura, no progress for 10+ minutes**
Most likely a corrupted ISO. Go back to step 6 and re-verify the checksum. If it doesn't match, redownload.

**"Apple macOS" doesn't show up as a guest OS option**
The Unlocker didn't apply correctly. Close VMware, re-run the unlocker executable, reopen VMware.

**VM won't power on / errors immediately**
Check for a SATA node conflict. Your two hard disks and CD/DVD drive all need distinct node numbers.

**Installer copies files then reboots into a blank EFI menu instead of continuing**
This is usually normal. OpenCore briefly shows its boot picker on every restart during the install loop and should auto-select the installer after a few seconds. Let it run.

**Xcode won't install from the App Store**
Expected. Use the direct `.xip` link in step 9 instead.

---

Nice job, hopefully it works.
