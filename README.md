# LSI SAS HBA 6Gbps Crossflash tools and guide in 2023 for LSISAS2008 (Fujitsu D2607, Dell H310)

This is a guide and toolset to crossflash LSI SAS2 (6Gbps) HBA controllers with newer IT or IR firmwares. Crossflashing is when we use a firmware made for another device from another brand. This is necessary because for a lot of hba controllers the manufacturer stopped firmwares updates (eg, fujitsu), so then it's necessary to use firmwares made for other hba controllers but using the same LSISASXXXX chipset.

This guide is focused on the LSISAS2008 chipset, but it should be applicable to any LSISAS2 chipset as long as you get the adequate SBR (or make your own) and firmware.

This repository contains all the files you need to put on a USB drive to crossflash any LSI SAS card (theoretically).

Also, this repository includes a new file you will not find anywhere else: `sas2haxp20.efi`, which is the hack of Marcan to skip vendor verification (Mfg page 2 verification failure) but applied on sas2flash p20.00.00.00 (there is no sas2flash p20.00.07.00, only the firmware can have this version, not the flashing tool), whereas Marcan's original sas2hax was from sas2flash p19. In my own tests it did not change anything in the crossflashing process but maybe it can help in some cases.

Note that crossflashing SAS1 (3Gbps) and SAS3 (12 Gbps) HBA cards is outside the scope of this guide, but you can find the official `sas3flash.efi` tool [here](https://www.broadcom.com/support/knowledgebase/1211161501344/flashing-firmware-and-bios-on-lsi-sas-hbas). Note that SAS3 12Gbps hard drives are compatible with SAS2 6Gbps HBA controllers, and inversely SAS2 drives work on SAS3 controllers. But IIRC SAS1 drives and controllers are incompatible with both SAS2 and SAS3. 

## The step-by-step guide to crossflashing any LSI SAS HBA card

### Preparing the crossflashing tools
1. Before starting, you need to know:

	a. your hba card id and brand (eg, Fujitsu D2607), this will define what SBR to use,
	b. the LSISASXXXX chipset on your HBA card, this will define which firmware crossflash, eg, LSISAS2008.
	c. the LSI HBA controller with the same chipset, eg, LSI 9211 has the LSISAS2008 chipset,
	d. how many ports your HBA controller has: if 4 ports, you need a 4i firmware, otherwise if 8 ports (more common case), you need a 8i firmware. Only 8i firmwares are included in this repository, you need to download other firmwares from Broadcom (ex-LSI and ex-Avago Technologies), Dell, or Supermicro.

2. Use [Rufus](https://github.com/pbatard/rufus/releases) to format and make a usb flash drive bootable with **FreeDOS**.
3. Copy content of my repo. See the section "Files sourcing" to know the source of each file.

At this point, the usb drive is bootable both into freedos and uefi shell v1! You can select from the BIOS boot menu (see the next sections).

### HBA controller reprogramming

From this point on, you do not need to reformat the usb drive. If you lack some firmwares, you can always just copy them, and restart all the steps in this section, but you do not need to prepare the USB drive again (no reformatting!).

Also, know that LSI HBA controllers, are VERY hard to brick, so feel free to experiment: as long as `megarec.exe` can see your controller, you are fine!

#### SBR hacking

1. Reboot and from the very first screen right away start pressing F12 repetitively for a few minutes until a boot selector appears. If this does not work, reboot and enter BIOS, usually by pressing F2, then it can offer a boot selector on the last tab. 
2. Boot into normal (non uefi) option first, freedos. If error memory corrupted, disconnect sas and sata drives that are bigger than 2TB (even if your current sas firmware supports drives bigger than 2TB! Freedos doesn't!). 
3. `megarec.exe -readsbr 0 origsbr.bin` - note: do this only the first time, to save your original sbr in case it's a rare card with special flags necessary to flash it and to detect hard drives slots (such as fujitsu). If you redo the whole flashing process later, skip this to avoid overwriting your original! 
	* CRITICAL NOTE: make sure to make a backup ASAP outside of the USB key, not because of the SAS address as can be read in some places (we do not care about it because a random address can be used instead, it’s like a MAC address, you just need it to be unique on your system to avoid conflicts), but to backup the whole SBR as it may contain special flags that are unique to your HBA controller, especially if it’s rarely used by other users online!
4. `megarec.exe -writesbr 0 sbr-a11.bin` or `sbr-a21.bin` for fujitsu, sbr-empty.bin for others. This sets a new sbr to allow updating to a newer firmware from another brand, we here use hacked sbr mixing data from other brands with a few flags specific to our own original brand (that's why the original sbr can be necessary, in case you need to make a custom sbr yourself as explained here, see below). Note: in FreeDOS, writing filenames and commands in uppercase or lowercase does not matter. 
5. `megarec.exe -cleanflash 0` to clean the firmware (while keeping the sbr). Note: i think this is equivalent to doing `sas2flash.efi -o -e 7` in the uefi shell, but not 100% sure. 

#### Crossflashing the firmware of another hba controller

6. Restart machine, tap f12 or use bios to boot into UEFI usb drive option to get UEFI shell.
7. Dont forget map -b to list drives, and identify the Removable Storage one for USB. Type `fsx:` where x is the number, usually `fs0:`
8. Do NOT do any other command between the sas2flash/sas2hax ones that are advised here, eg, do NOT try to sas2flash -listall in the middle, it may make the rest failbfor some reason as reported by some, sas2flash seems to be finicky and require a precise sequence of actions to work for crossflashing (which is not its primary purpose)! 
9. `sas2haxp20.efi -o -f 2118IT.BIN -b mptsas2.rom` for fujitsu, sas2flash.efi for others. No need for sas2flash -o -e 6 or 7 beforehand but you can try if you messed up before. Use 2118IT.BIN for IT mode, 2118IR.BIN for IR mode (hardware RAID). IT mode is better if you fant to make a software RAID, and in general it's better for files recovery if you ever get a disk corruption (which WILL happen eventually for any hard drive or storage medium). 
10. With fujitsu cards, the whole process should almost complete fine (Firmware download successful, no Mfg page 2 verification failed), but will fail at resetting the adapter with Fault code: 704. This is normal, your card is almost done but not fully yet, you need to reboot and redo the same. It's really necessary to reboot, trying any command in the current shell without rebooting will make the flashing fail and you may have to restart all the steps above. 
11. For fujitsu only: Reboot into UEFI shell again (even though you were in it already), and write the same command again: `sas2haxp20.efi -o -f 2118IT.BIN -b mptsas2.rom` - this time it should complete fully, including the adapter reset! ![IMG_20230613_223746](IMG_20230613_223746.jpg)
12. Optional, only necessary if you plan on having multiple SAS HBA card simultaneously on the same motherboard (think about the future) : `sas2haxp20 -o -sasadd 5000xxxxxxxxxxxx` where x are 12 hexadecimal characters, just to make the card unique. You can restore your old id or just make one up. Eg of a basic one you can make up: `5000123456789abc`
13. Do `sas2hax -listall` to check, you should see your card is up! In case you already rebooted, you can always check later by rebooting into UEFI shell. 
14. Optional, or necessary for fujitsu if you did not use the right sbr (eg, some drives are not detected): Reboot to freedos (no uefi), and do `megarec -writesbr sbr-a11.bin` or a21 for fujitsu, for others no need to, then reboot one last time. If you already used one of these two sbr at the start and you can see all your drives detected at boot POST, then you do not need to do this step. If you cannot see all the drives detected, then try another sbr, no need to reflash the firmware. You can try as many sbr as you want without habing to reflash your hba card with sas2hax, once the firmware is there, it will stay unless you do megarec -cleanflash! 
15. If you forgot to include the rom, you won't be able to use sas drives as bootable devices (ie, cannot install an OS on them). But you can always flash the bootable rom later, by rebooting into the UEFI shell and type `sas2haxp20 -o -b MPTSAS2.ROM` (without the firmware 2118IT.BIN).
16. Reboot, and you should see a new POST setup screen for the hba controller and it should show it detects all drives now! ![IMG_20230613_225013](IMG_20230613_225013.jpg)

### A few helpful rules and tips

1. Rule: to execute EXE files such as megarec.exe, use non uefi freedos env. For efi files, boot into tianocore edk2 v1 uefi shell env. If you let the drive boot without pressing f12, it will boot into freedos by default.
2. Use TAB key to autocomplete commands and filenames, but not flags. Can tap TAB multiple times when the first offered option is not the one you are looking for. 
3. After first flash with sas2hax, will get a Fault 704 error. Don't worry, reboot and retry, it should then work this time.
4. Sas2hax necessary for fujitsu cards. Can also be used for other cards to bypass mfg branding verification, but still the error may appear, and it is not necessary for dell. Also, sas2hax is based on sas2flash p19, not p20, so it's slightly outdated. Although sas2hax can and is used in the current tutorial to flash a p20.00.07.00 firmware. 
5. In the tuto, dashes are missing, but otherwise it works! writesbr and sasadd https://forums.unraid.net/topic/12114-lsi-controller-fw-updates-irit-modes/page/47/
6. If unsure if, a21 or a11, no worries, can change at the end and change sbr multiple times with megarec -writesbr, just do not cleanflash, or you will have to redo sas2hax. 
7. In bios set sata in ide mode, not raid! Then should see AvagoTech POST setup at boot and listing of drives. If installed boot rom. 
8. If you need files such as firmwares (eg supermicro) that are specific to your hba card, just copy them over, you do NOT need to reformat with rufus at any time. This will avoid the risk that you overwrite and lose your original SBR as happened to me! 
9. ALWAYS BACKUP YOUR ORIGINAL SBR IN CASE YOU NEED TO MAKE A CUSTOM SBR! In case you have a common HBA card, it's not that, bad, because you can use the SBR given by others online, but if you have a very specific card with a custom SBR, using others sbr may make the card half functional, such as not showing all drives. In this case, you need to take a sbr from a brand that accepts a newer firmware such as dell, and hack is with HxD (a hexadecimal editor) to apply the special flags from your original sbr, effectively making a hybrid custom sbr, as shown in the tuto, and then flash it on your card using megarec before attempting sas2haxp20 again. https://marcan.st/2016/05/crossflashing-the-fujitsu-d2607/
10. To crossflash, need to know chipset version, lsisasxxxx. Why crossflash? To be able to get a fismware of higher version to support drives bigger than 2tb.
11. Do not worry about mistakes, apart from making sure to keep your original sbr in case you have a very rare hba card that uses different flags, there should be no risk to brick your card, or at least nobody really could. As long as megarec can see and flash your card, you can reconfigure it. Just restart the steps above from the start. 
12. Do NOT use firmware of phase 20 below revision 07, they were buggy as hell and could cost you hard drives data! So either install phase 19 or p20.00.07.00, but nothing between. https://gist.github.com/dreamcat4/f9e16100d68be759ebaa0c9403a55121

### References

Here are several resources that I read and that helped me learn how to crossflash, you may be interestet in reading them for more information or background, although no other guide covers as many lsi cards as here. 

If a link is dead, try to use the Wayback Machine, I tried to backup all the relevant links there, including executables wherever possible. 

1. https://gist.github.com/dreamcat4/f9e16100d68be759ebaa0c9403a55121
2. P20.00.07.00 latest firmware (also mention that phase = version) https://docs.broadcom.com/docs/12350530
3. https://www.broadcom.com/support/knowledgebase/1211161501344/flashing-firmware-and-bios-on-lsi-sas-hbas
4. https://marcan.st/2016/05/crossflashing-the-fujitsu-d2607/
5. https://forums.unraid.net/topic/12114-lsi-controller-fw-updates-irit-modes/page/46/#comment-522038
6. The instructions for Fujitsu D2607 were heavily inspired by FingerlessGloves’ guide (and I tested myself several variants to check that yes indeed the steps i retained are necessary): https://forums.unraid.net/topic/12114-lsi-controller-fw-updates-irit-modes/?do=findComment&comment=528319

### Troubleshooting

1. If sas2flash.efi fails because "No LSI SAS adapter found" error in uefi shell, then need to remove the branding first! Using megarec.exe in FreeDOS with boot option 4. See tuto: https://www.truenas.com/community/threads/ibm-serveraid-m1015-and-no-lsi-sas-adapters-found.27445/ and https://gist.github.com/dreamcat4/f9e16100d68be759ebaa0c9403a55121
2. To boot into UEFI: first ensure SecureBoot is disabled and UEFI booting from usb is enabled in BIOS (press F2 at startup), then make a bootable usb uefi drive using rufus and https://github.com/pbatard/UEFI-Shell , then replace the uefi files with tianocore edk2 version 1 (instead of v2) from https://superuser.com/questions/1057446/how-do-i-boot-to-uefi-shell , then copy sas2flash.efi and the IT firmware and bootable rom for the HBA SAS controller card from https://www.truenas.com/community/threads/how-to-flash-lsi-9211-8i-using-efi-shell.50902/ and follow instructions in this last link, to boot into uefi shell from usb drive and then use sas2flash.efi to flath to it mode.
3. To boot into uefi shell, use tianocore EDK2 Version 1 (not version 2 because incompatible with sas2flash.efi, too old), and can create a Bootable usb drive and newer link to uefi shell tianocore https://superuser.com/questions/1057446/how-do-i-boot-to-uefi-shell
4. How to flash uefi lsi sas2008 cards using sas2flash.efi and how to boot into uefi shell v1 + select firmware for adequate number of portsk(8i = 8 ports, 4i = 4 ports) + bootrom is necessary if we want to boot from sas drives! https://www.truenas.com/community/threads/how-to-flash-lsi-9211-8i-using-efi-shell.50902/
5. Network address as long as you don't have the same addresses connected to the same machine it wont be a problem. + nearly impossible to brick so don't fret https://forums.servethehome.com/index.php?threads/need-help-on-dell-h310.24708/ - eg 500123456789abcd
	* Alternative: can generate any set of 12 random hexadecimal characters as the address after 5000, eg, 5000d847203de496 . https://forums.unraid.net/topic/12114-lsi-controller-fw-updates-irit-modes/page/47/
6. Need specific sbr-a11.bin or sbr-a21.bin for flashing to work, otherwise need to use Dell firmware and then p7 before p20: https://forums.unraid.net/topic/12114-lsi-controller-fw-updates-irit-modes/page/46/#comment-522038 and https://www.truenas.com/community/threads/fujitsu-9211-8i-d2607-lsisas2008-wont-flash-to-anything.48946/page-2 and https://forums.unraid.net/topic/12114-lsi-controller-fw-updates-irit-modes/?do=findComment&comment=528319
7. If you have a Supermicro card, use their firmwares, they are very specific
8. Use mpsutil to show current firmware version https://www.truenas.com/community/threads/lsi-sas2008-hba-aka-9211-8i-q-firmware-upgrade-to-stop-bios-detection-fix-hotswap-not-working.84689/
9. If you are stuck at the stage of "no controller detected, need firmware" and you type in the firmware binary, it's not going to work. Even if with sas2flash -o -e 7 it appears to work at first, it will only use the firmware to know how to flash the hba, but then you need to download the firmware on it using sas2flash -o -f 2118IT.bin and then it will again complaint that no card is detected, and then if you again try to input the firmware 2118IT.BIN, it will start and then fail to download the firmware. Normally if you did things right for your hba, this should never happen, the commands should work right away without sas2flash complaining about no controller detected. If that'r the case, it means you need to upload another sbr for your hba using megarec.

### Files sourcing

The files in this repository were sourced from the following sources:

1. [D2607.zip](https://forums.unraid.net/topic/12114-lsi-controller-fw-updates-irit-modes/page/46/#comment-522038) ([direct download](https://mega.nz/#!T8cSRSwL!UUo72yqq-ov2ulKgaznP8vgVeE_zBMdpBQ7ZB2LvfO8)) by FingerlessGloves, which includes the latest [phase 20 v20.00.07.00 firmware for the LSI 9211-8i HBA controller](https://docs.broadcom.com/docs/12350530) as well as megarec.exe (which itself likely comes from the [m1015.zip file shared here](https://www.truenas.com/community/threads/where-is-the-megarec-utility-to-crossflash-m1015-to-it-mode.25130/)).
2. [Toolset_PercH310_H200_2_LSI9211_P20.00.07.00_efimod.zip](https://forums.unraid.net/topic/12114-lsi-controller-fw-updates-irit-modes/?do=findComment&comment=527688) ([direct download](https://www.mediafire.com/?9cbklh4i1002n23)) by Fireball3, with all the tools and firmwares necessary to crossflash Dell H310.
3. The UEFI shell files are from the [Tianocore EDK2](https://github.com/tianocore/edk2) project, which is an open-source implementation of an UEFI shell. I took the [EDK2 v1 binaries](https://github.com/tianocore/edk2/tree/UDK2018/EdkShellBinPkg/FullShell), as otherwise sas2flash fails in UEFI v2 shells.
4. The UEFI files were organized according to the [ISO](https://github.com/pbatard/UEFI-Shell) provided by pbatard, the same author as Rufus, but his ISO uses EDK2 v2, whereas we need v1.
4. The [lsirec](https://github.com/marcan/lsirec) tool, to be used under Linux, is from Marcan, the same author who made `sas2hax.efi`.
5. `sas2haxp20.efi` is my own implementation of [Marcan’s hack of sas2flash.efi into sas2hax.efi](https://marcan.st/2016/05/crossflashing-the-fujitsu-d2607/), but applied on sas2flash.efi p20 instead of p19 as done by Marcan. Hence, this file is an exclusivity of this repository.
* `sas2hax.efi` is the [original hack by Marcan](https://marcan.st/2016/05/crossflashing-the-fujitsu-d2607/) to bypass the vendors verification (Mfg page 2 verification failure).
6. `sas2flash.efi` is the original p20 tool from [Broadcom/LSI](https://www.broadcom.com/support/knowledgebase/1211161501344/flashing-firmware-and-bios-on-lsi-sas-hbas).

### Credits and license

This guide (README.md) is under Creative Commons CC-BY-SA 4.0, written by Stephen Karl Larroque in 2023.

I claim no credit for all the other files, please see their respective licenses. I believe this package fits under Fair Use for preservation purposes as the vendors do not provide support for these controllers anymore. 
