![logo](https://imgur.com/YEZO0rg.png)

# What is OpenCore?

OpenCore is an alternative bootloader to CloverEFI or Chameleon. It is not only for Hackintosh and can also be used on real macs for purposes that require an emulated EFI. It also aims to have the ability to boot Windows and Linux. It has a clean codebase and aims to stay closer to how a real mac bootloader functions. Kext injection has been greatly improved. While already functioning well, OpenCore should be considered in alpha stage at this time and is intended to be used by experienced hackintosh users, developers, or users who are happy to recover a system which fails to boot or becomes broken in some way.

**This guide may not always be able to keep up with every change to OpenCore,** 
**(currently OpenCore is in active development,and therefore a moving target)** 
**please keep that in mind when compiling the latest version of OpenCore.**
**To be safe, use release versions of OpenCore rather than the latest commits.** 

This guide is intended to complement the excellent opencore "configuration.pdf" rather than be used instead of it. If you did not already do so, please read it now: 
[Full manual](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/Configuration.pdf)


# Current known issues
* Z390 based systems require workarounds to non working NVRAM. [Emulated-NVRAM](https://github.com/MacProDude/Emulated-NVRAM)
* Refer to opencore bugtracker for current known bugs [here](https://github.com/acidanthera/bugtracker/issues)

# Getting Started

Requirements:

* [OpenCorePkg](https://github.com/acidanthera/OpenCorePkg/releases) (Advanced users can build the latest from source code, less advanced users should stick to the builds on the release page).
* [AppleSupportPkg](https://github.com/acidanthera/AppleSupportPkg/releases)
* [AptioFixPkg](https://github.com/acidanthera/AptioFixPkg/releases)
* [WhateverGreen](https://github.com/acidanthera/WhateverGreen/releases)
* [Lilu](https://github.com/acidanthera/Lilu/releases)
* [VirtualSMC](https://github.com/acidanthera/VirtualSMC/releases) *FakeSMC is (in this guide) not recommended.*
* [Emulated-NVRAM](https://github.com/MacProDude/Emulated-NVRAM) *For emulated Nvram if you have nvram issues.*
* Xcode (or other plist editor) to edit .plist files.
* USB drive formatted as MacOS Journaled with GUID partition map. This is to test opencore without overwriting your working Clover.
* Knowledge of how a hackintosh works and what files yours requires.
* A previously setup and functioning hackintosh is assumed. * Which you are happy to potentially break *
* Time and patience. Without these, you are wasting your effort.
* [InsanelyMac-Discord](https://discord.gg/EdmdKRW) If you need any extra help join our discord.


# Creating the USB

Creating the USB is simple, format a USB stick (any size will suffice) as MacOS Journaled with GUID partition map. 

![Formatting the USB](https://i.imgur.com/TAyeeGd.png)

Next, mount the EFI partition on the USB with either diskutil terminal command or Clover Configurator.

![mountEFI](https://i.imgur.com/yCWBGoJ.png)

By default, the EFI partition will be empty.

![Empty EFI partition](https://i.imgur.com/4iLK9Gd.png)

# EFI folder structure

To setup OpenCore’s folder structure, you’ll want to grab those files from OpenCorePkg and construct your EFI to look like the one below:

![base EFI folder](https://i.imgur.com/4ZE7jYj.png)

Place your necessary .efi drivers from AppleSupportPkg and AptioFixPkg into the *drivers* folder and kexts/ACPI into their respective folders.

Here's what mine looks like:

![Populated EFI folder](https://i.imgur.com/rrJ0Nc4.png)

# Setting up your config.plist

While sharing the name, the config.plist in OpenCore, is very different from Clover config.plist, they cannot be mixed and matched. It is also not recommended to duplicate every patch and option from your clover config. 

First, duplicate the `sample.plist`, rename it to `config.plist` and open in your .plist editor of choice.

![Base Config.plist](https://i.imgur.com/oDGVALF.png)

The config contains a number of sections:

* **ACPI:** This is for loading, blocking and patching the ACPI.
* **DeviceProperties:** This is where you'd inject PCI device properties or set Intel Framebuffer.
* **Kernel:** Where we tell OpenCore what kexts to load, what order to load and which to block.
* **Misc:** Settings for OpenCore's boot loader itself.
* **NVRAM:** This is where we set certain NVRAM properties like boot flags and SIP.
* **Platforminfo:** This is where we setup your SMBIOS.
* **UEFI:** UEFI drivers and related options. 

We can delete *#WARNING -1* and  *#WARNING -2* You did heed the warning didn't you?

# ACPI

**Add:** Here you add your SSDTs or custom DSDT. (SSDT-EC.aml for example)

**Block**: Certain systems benefit from dropping some acpi tables, most modern desktops however require nothing in this section.

**Patch**: In OpenCore we should be keeping ACPI device renames to a minimum as they are often harmful and unnecessary. If your system absolutely needs something, you should add it in this section. Refer to configuration.pdf.

* For example, common device renames are handled now by WhateverGreen on-the-fly and in a safer way:
- GFX0 to IGPU
- HECI to IMEI
* Do NOT do these in the config.plist nor in DSDT/SSDT.

* Do NOT rename EC0 to EC as this can cause an incompatible kext (AppleACPIEC) to load and cause strange issues at any time or a non bootable system.

 * For X299 systems only:
- PC00 to PCI0 : Since the PC00 device has many PCI devices, recommend using Patch methods than rename using SSDT.
- LPC0 to LPCB : Same as changing method in PC00 to PCI0
- CPxx to PRxx : No need to rename CPU name. (does not affect sleep and wake up)
- EC0 to EC : No need to rename. To refer to this document 
[SSDT-EC-USBX](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/AcpiSamples/SSDT-EC-USBX.dsl)

**Quirks**: Certain ACPI fixes. Avoid unless necessary.

* **FadtEnableReset:** NO (Enable reboot and shutdown on legacy hardware, not recommended unless needed).
* **NormalizeHeaders:** Cleanup ACPI header fields, irrelevant in 10.14.
* **RebaseRegions:** Attempt to heuristically relocate ACPI memory regions.
* **ResetHwSig:** Needed for hardware that fail to maintain hardware signature across the reboots and cause issues with
waking from hibernation.
* **ResetLogoStatus:** Workaround for systems running BGRT tables.

![ACPI](https://i.imgur.com/WSa88oU.png)

&#x200B;

# DeviceProperties

**Add**: Injects Device properties.

`PciRoot(0x0)/Pci(0x2,0x0)` -> `AAPL,ig-platform-id`

* Sets integrated graphics framebuffer, insert required value.  Don't forget to add Stolemem and patch-enable if necessary.

`PciRoot(0x0)/Pci(0x1b,0x0)` -> `Layout-id`

* Injects Audio device layout id, insert required value from AppleALC documentation [here](https://github.com/acidanthera/AppleALC/wiki/Supported-codecs).

**Block**: Removes device properties from map. Normally not required.

![DeviceProperties](https://i.imgur.com/TEKKxPf.png)

# Kernel

**Add**: Here we can specify kexts to inject from our EFI into the kernel kextcache. 
Order of kexts is important, they are loaded in this order. Plugins for other kexts should always come after the main kext. Lilu should be first, then Lilu plugins like WhateverGreen and VirtualSMC. 

**Emulate**: Needed for spoofing CPU, for unsupported CPUs.

* CpuidMask: When set to zero, original CPU bit will be used.
* CpuidData: The value for the CPU spoofing, hex swappped.

**Block**: Blocks kexts from loading. Sometimes needed for disabling Apple's trackpad driver for some laptops.

**Patch**: Kext or kernel patches can be added here. 

**Quirks**:

* **AppleCpuPmCfgLock:** Only needed when CFG-Lock can't be disabled in BIOS. Avoid unless necessary.
* **AppleXcpmCfgLock:** Only needed when CFG-Lock can't be disabled in BIOS. Avoid unless necessary.
* **AppleXcpmExtraMsrs:** Disables multiple MSR access needed for unsupported CPUs.
* **CustomSMBIOSGuid:** Performs GUID patching for UpdateSMBIOSMode Custom mode. Usually relevant for Dell laptops.
* **DisableIOMapper:** Preferred to dropping DMAR in ACPI section or disabling VT-D in bios.
* **ExternalDiskIcons:** External Icons Patch, for when internal drives are treated as external drives
* **LapicKernelPanic:** Disables kernel panic on AP core lapic interrupt. Often needed on HP laptops.
* **PanicNoKextDump:** Allows for reading kernel panics logs when kernel panics occurs.
* **ThirdPartyTrim:** Trimforce would be preferred via terminal, as most 3rd party Nvme and SSD's are now supported.
* **XhciPortLimit:** This the 15 port limit patch, use only while you create a usb map (ssdt-uiac.aml) or injector kext. Its use is NOT recomended long term.

![Kernel](https://i.imgur.com/vQqn5eo.png)

# Misc

**Boot**: Settings for boot screen.
* **Timeout:** This sets how long OpenCore will wait until it automatically boots from the default selection.
* **ShowPicker:** If you need to see the picker screen, you better choose YES.
* **UsePicker:** Want to boot with opencore? must choose yes.
* **Target:** Setting for logging type (by default logging output is hidden).
* **HideSelf:** If you want to hide EFI partion on OC Bootloader choose YES.
* **HibernateMode:** Recommended set to None.
* **ConsoleBeHaviousOs:** Set to ForceGraphics for most systems.
* **ConsoleBehaviousUI:** Set to Text for most systems.

** You won't be able to boot with Open Core Bootloader If you do not set **YES** at UsePicker.
** If you want to make macOS the default boot disk, set 'System Preferences > Startup Disk > (Your preferred OS disk)' as the default boot disk.

**Debug**:
* **DisableWatchDog:** (May need to be set to yes if macOS is stalling while logging to file is enabled).
* **Target:** Logging level. 75 enables full logging to screen and file. (find log file on root of EFI partition).
0 fully disables boot log. (Debug Release of OpenCore maybe required for BugTracker with -keepsyms as boot-arg).

**Security**:
* **RequireSignature:** See detailed explanation in configuration.pdf.
* **RequireVault:** For now choose NO.
* **ScanPolicy:** Allows customization of disk and file system types which are scanned (and shown) by opencore at boot time.

**Tools**: Used for running boot time tools like clearing NVRAM, EFIShell or memtest86. Enable if required.

![Misc](https://i.imgur.com/tpr1OcL.png)

# NVRAM

**Add:**

**7C436110-AB2A-4BBB-A880-FE41995C9F82 (APPLE_BOOT_VARIABLE_GUID)**

* **boot-args:** -v debug=0x100 keepsyms=1 , etc (Boot flags)
* **csr-active-config:** <00000000> **(Settings for SIP, recommended to manully change this within Recovery partition with csrutil and preferred to leave as '00000000' in the config.plist file).**
   * `00000000` - SIP completely enabled
   * `30000000` - Allow unsigned kexts and writing to protected fs locations
   * `E7030000` - SIP completely disabled
* **nvda_drv:**  <> (For enabling Nvidia WebDrivers, set to 31 if running a Maxwell or Pascal GPU. This is the equivalent to setting nvda_drv=1 but instead we convert it from text to hex.
* prev-lang:kbd: <> (Needed for non-latin keyboards) If you find Russian, you didnt read the manual...

**4D1EDE05-38C7-4A6A-9CC6-4BCCA8B38C14 (APPLE_VENDOR_VARIABLE_GUID)**
* UIScale : Boot time screen resolution. May need to be set to 02 to enable HiDPI scaling in FileVault 2 UEFI password interface and boot screen logo. but using a 10, you can see the big apple logo with HiDPI.
* This will fail when console handle has no GOP protocol. When the firmware does not provide it, it can be added with ProvideConsoleGop UEFI quirk set to 'YES´and in protocols section ´ConsoleControl´to YES.

**Block**: Forcibly rewrites NVRAM variables, not needed for us as `sudo nvram` is prefered but useful for those edge cases.

**LegacyEnable:** Allows for NVRAM to be stored on nvram.plist for systems without working NVRAM. (Example Z390).

**LegacySchema:** Used for assigning nvram variable on such systems. (This is written to the NVRAM.plist).

[For futher information see the Emulated NVRAM Guide](https://macprodude.github.io/Emulated-NVRAM/)

![NVRAM](https://i.imgur.com/aQp4Ya2.png)

# Platforminfo

**Automatic**: NO (setting YES will provide default values from the Generic section, which in some cases may be acceptable).

**Generic**:

* **SpoofVendor:** YES (This prevents issues with having "Apple,inc" as manufacturer).
* **SystemUUID:** Can be generated with MacSerial or use previous from Clover's config.plist.
* **MLB:** Can be generated with MacSerial or use previous from Clover's config.plist.
* **ROM:** <> (6 character MAC address, can be entirely random but should be unique).
* **SystemProductName:** Can be generated with MacSerial or use previous from Clover's config.plist.
* **SystemSerialNumber:** Can be generated with MacSerial or use previous from Clover's config.plist.

**DataHub**:
Fill all these fields to match your clover smbios.

**PlatformNVRAM**:
Fill all these fields to match your clover smbios.

**SMBIOS**:
Fill all these fields to match your clover smbios.

**UpdateDataHub**: YES (Update Data Hub fields)

**UpdateNVRAM**: YES (Update NVRAM fields)

**UpdateSMBIOS**: YES (Update SMBIOS fields)

**UpdateSMBIOSMode**: Create (Replace the tables with newly allocated EfiReservedMemoryType)

![PlatformInfo](https://i.imgur.com/icsZ1BD.png)

# UEFI

**ConnectDrivers**: YES

**Drivers**: Add your .efi drivers here. (HFSPlus, AptoMemoryFix, APFSLoader, etc)

**Protocols**:

* **AppleBootPolicy:** (Ensures APFS compatibility on VMs or legacy Macs).
* **ConsoleControl:** Needed on most APTIO firmwares otherwise you may see text output during booting instead of Apple logo.
* **DataHub:** (Reinstalls Data Hub).
* **DeviceProperties:** (Ensures full compatibility on VMs or legacy Macs).

**Quirks**:

* **ExitBootServicesDelay:** 0 (Switch to 5 if running ASUS Z87-Pro with FileVault2).
* **IgnoreInvalidFlexRatio:** Required for almost all pre-skylake based systems.
* **IgnoreTextInGraphics:** (Fix for UI corruption when both text and graphics outputs happen).
* **ProvideConsoleGop:** (needed when GPU doesn't have a GOP Firmware/Driver (Also check for CSM in BIOS and is Disabled).
* **ReleaseUsbOwnership:** (Releases USB controller from firmware driver).
* **RequestBootVarRouting:** (Recommended to be enabled on all systems for correct update installation, Startup Disk control panel functioning, etc.
* **SanitiseClearScreen:** (Fixes High resolutions displays that display OpenCore in 1024x768) Also necessary on select AMD GPUs on Z370.

![UEFI](https://i.imgur.com/tWJllin.png)


# Time to boot with OpenCore!

![AboutThisMac](https://i.imgur.com/BjQwhPC.png)

# Making Opencore your default Bootloader

When you are happy opencore boots your system correctly, simply mount your Clover efi partition, (back it up somewhere safe) and overwrite it with your OpenCore one. Certain system BIOS may require you to manually remove Clover as an EFI boot option (and extra special system might need a factory reset to permanently remove it).


# Credit
* [Apple](https://www.apple.com) for MacOS.
* [Acidanthera](https://github.com/acidanthera) for everything they contribute to hackintosh.
* [vit9696](https://github.com/vit9696) for OpenCore.
* [khronokernel](https://github.com/khronokernel) for the original guide. 
* [Pavo-IM](https://github.com/Pavo-IM) for persistant corrections. :D
* [ZISQO](https://github.com/zisqo) to translate this guide for korean language.
* [MacProDude](https://github.com/MacProDude) for Images and Corrections.
