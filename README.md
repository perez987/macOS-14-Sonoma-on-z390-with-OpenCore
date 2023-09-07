## macOS 14 Sonoma beta on Z390 Aorus Elite using OpenCore

<p align="center">
<img width="256" src="Sonoma icon.png">
</p>

### Preface

macOS 14 Sonoma is still in beta. Its official presentation is scheduled for a few weeks, so it is to be expected that there will be no major changes between now and the final version.

I have been testing all the beta versions released by Apple and I have found a very stable system that, logically, has improved with each version but that, from the first one, allowed me to work with it in a way close to that of a daily use system.
Sonoma required fewer changes to OpenCore and kexts than were necessary to install older systems such as Big Sur, which was a big challenge for developers. This time, minor changes allowed Sonoma to be installed almost immediately after the first beta version was released.

Of course, there have been problems to work hard on, not all solved as of today. For now I will note the loss of Wi-Fi with Broadcom chipsets used in Mac models before 2017 and in Fenvi PCI-e cards, widely used in Hacks.

### Hardware

* Motherboard Gigabyte Z390 Aorus Elite
* CPU Intel i9-9900K
* GPUs: iGPU Intel UHD 630 / AMD Radeon RX 6600 XT
* Audio Realtek ALC1220
* Ethernet Intel I219V7
* Wi-Fi + BT Fenvi FV-T919 (BCM94360CD).

### BIOS settings (F10h version)

* CFG Lock: Disabled
* CSM: Disabled
* VT-d: Disabled
* Fast Boot: Disabled
* OS Type: Windows 8/10 WHQL
* Platform Power Management: Disabled
* XHCI Hand-Off: Enabled
* Network Stack: Disabled
* Wake on LAN: Disabled
* Secure Boot: Disabled
* Integrated Graphics: Enabled
* DVMT Pre-allocated: 256M o higher.

### What works well?

* dGPU AMD as main card
* iGPU in headless mode
* Shutdown, restart and sleep
* Ethernet
* Sound (also HDMI)
* USB ports (USB port map for this board)
* Bluetooth Fenvi T919.

### What's not working?

Fenvi T919 Wi-Fi: macOS Sonoma has dropped support for all Broadcom Wi-Fi present on Macs before 2017. Fenvi T919 and HB1200 have BCM4360 chipsets (not supported) so Wi-Fi does not work in Sonoma. Bluetooth works fine. This is a serious inconvenience because functions related to the Apple ecosystem (Airdrop, Continuity Camera, etc.) are also lost. A fix is proposed later.

---

### Installing macOS Sonoma

I have updated macOS Ventura to Sonoma but creating USB boot media to install from scratch is another option for those who prefer to do it that way.

* You must have macOS 13.4 Ventura or later
* System Settings >> Software Update >> Beta Updates >> click on the info icon
* Options: Disabled / macOS Sonoma Public Beta / macOS Sonoma Developer Beta
* Choose macOS Sonoma Developer Beta or macOS Sonoma Public Beta.

If you are on macOS 13.3 Ventura or earlier, it is recommended to go to 13.4 before upgrading to Sonoma. If not, the way to enable the beta updates channel is different: you have to get *macOSDeveloperBetaAccessUtility.dmg* from the download site at *developer.apple.com* and, when you run it, beta updates option is added in System settings.

To create the USB installation media so you can install Sonoma from scratch:

- Get the complete installation package from Apple's servers. I use the app [Download Full Installer ](https://github.com/perez987/DownloadFullInstaller)(original by _scriptingosx_), main window shows all versions available for download from Big Sur to Sonoma. To show beta versions, go to Settings >> SeedProgram and choose DeveloperSeed or PublicSeed
- The package is downloaded as InstallAssistant-14.0-build.number.pkg, double click on the package to generate Install macOS Sonoma beta.app in the Applications folder
- Format a USB stick of at least 16Gb with GUID partition scheme and Mac OS Plus (journaled) format, name it (e.g. USB)
- Open Terminal and run this command
`sudo /Applications/Install\ macOS\ beta.app/Contents/Resources/createinstallmedia --volume /Volumes/USB --no-interaction`

<br>
<p align="center">
<img width="512" src="DownloadFullInstaller.png">
</p>
<br>

At the end, you can reboot from the USB device and begin Sonoma installation.

Once the final version of Sonoma is released, users will be able to download it from the Apple Store and it will also be notified as an available update (not beta) in System Settings.

### OpenCore and EFI folder

Update OpenCore and kexts to Sonoma compatible versions. OpenCore, at least version 0.9.4. Settings used with macOS Ventura may work with macOS Sonoma. Updating OpenCore and kexts, there are no significant changes to the config.plist file, which may be the same for both systems.

For the update to be successful, 2 parameters in config.plist related to security must be adjusted:

- `SecureBootModel=Default` or `x86legacy` (Apple Secure Boot as Default sets the same model as in SMBIOS and x86legacy is designed for SMBIOS that lack T2 chip and virtual machines)
- SIP enabled (`csr-active-config=00000000`.

It is advisable to have Gatekeeper enabled (`sudo spctl –master-enable` in Terminal).
Note: in last versions of Ventura, `sudo spctl –master-enable` (or disable) has been replaced by `sudo spctl –global-enable` (or disable). For now, both commands work fine.

These security options can be changed after installation as they are not required out of updating macOS.

### config.plist

I get best results with iMac19.1 SMBIOS and the iGPU enabled in BIOS.

These are the main details when configuring config.plist.

- ACPI: SSDT-EC-USBX.aml, SSDT-PLUG.aml and SSDT-PMC.aml. SSDT-AWAC.aml is not required on my system but, if in doubt, add it because it does not cause any harm if it is present without being needed
- ACPI >> Quirks: all = False

- Booter >> Quirks: AvoidRuntimeDefrag, DevirtualiseMmio, ProtectUefiServices, ProvideCustomSlide, RebuildAppleMemoryMap, SetupVirtualMap and SyncRuntimePermissions = True
- Booter >> ResizeAppleGpuBars=-1

- DeviceProperties >> Add
	- PciRoot(0x0)/Pci(0x2,0x0)
		- AAPL,ig-platform-id | Data | 0300913E
		- device-id | Data | 9B3E0000
		- enable-metal | Data | 01000000
		- rps-control | Data | 01000000
	- PciRoot(0x0)/Pci(0x1.0x0)/Pci(0x0.0x0)/Pci(0x0.0x0)/Pci(0x0.0x0)
		- unfairgva | Number | 6
	- PciRoot(0x0)/Pci(0x1F,0x3)
		- layout-id | Data | 07000000
	- PciRoot(0x0)/Pci(0x14,0x0)
		- acpi-wake-type | Data | 01
		- acpi-wake-gpe | Data | 6D

- Kernel > Add: Sonoma compatible kexts, Lilu.kext in the first place, UTBMap.kext specific for this motherboard
- Kernel >> Quirks: CustomSMBIOSGuid, DisableIoMapper, DisableIoMapperMapping, DisableLinkeditJettison, PanicNoKextDump and PowerTimeoutKernelPanic = True
- Kernel >> Quirks: SetApfsTrimTimeout = 0

- Misc >> Boot: HibernateMode=None, PickerAttributes=144, PickerVariant=Default, ShowPicker=True
- Misc >> Debug: AppleDebug, ApplePanic and DisableWatchDog = True, Target=3
- Misc >> Security: AllowSetDefault=True, BlacklistAppleUpdate=True, ExposeSensitiveData=6, SecureBootModel=x86legacy or Default

- NVRAM
	- WriteFlash=True
	- Add >> 7C436110-AB2A-4BBB-A880-FE41995C9F82:
		- boot-args >> agdpmod=pikera
		- csr-active-config >> 00000000
		- run-efi-updater >> No
	- Delete >> 7C436110-AB2A-4BBB-A880-FE41995C9F82:
		- boot-args and csr-active-config

- PlatformInfo
	- Generic >> iMac19.1
	- UpdateDataHub, UpdateNVRAM and UpdateSMBIOS = True
	- UpdateSMBIOSMode >> Custom

- UEFI >> Quirks: EnableVectorAcceleration and RequestBootVarRouting = True
- UEFI >> Quirks >> ResizeGpuBars=-1.

**Notes about software updates**

There are 3 SMBIOS that I can use on my PC: iMac19,1 / iMacPro1,1 / MacPro7,1. My favorite is iMac19.1. Regarding the updates that are notified in Software Update and the size of the update (full or incremental package), there are some conditions to take into account.

1. Getting Update notification

* iMac19.1 model (2019 iMac 27″) lacks a T2 security chip and, when using this SMBIOS model, you receive update notifications
* iMacPro1,1 (iMac Pro 27″, late 2017) and MacPro7,1 (Mac Pro 2019) models do have a T2 chip and, when using these SMBIOS models, you do not receive update notifications
* iMacPro1,1 and MacPro7,1 models receive update notifications if configured as vmm (virtual machine): `revpatch=sbvmm` in boot-args along with RestrictEvents.kext.

2. Size of the update (full or incremental)

* Systems that have not been patched by OCLP (or the patch has been reverted), whether the SMBIOS model has a T2 chip or not, require `revpatch=sbvmm` in boot-args along with RestrictEvents.kext for incremental updates. Without this setting, they get full upgrade packages
* All systems that have the OCLP root patch applied receive full-size updates.

In summary, using iMac19.1 you get update notifications but the updates are full-size.

After the system is updated, RestrictEvents.kext and the boot argument can be disabled because they are not required for normal Sonoma operation.

---

### Broadcom Wi-Fi stops working in Sonoma

Apple has dropped support for Broadcom Wi-Fi chipset drivers used in pre-2017 Macs:

Dependant of AirPortBrcmNIC.kext (IO80211FamilyLegacy.kext plugin)

* device-id pci12e4,43ba >> BCM43602
* device-id pci12e4,43e3 >> BCM4350
* device-id pci12e4,43a0 >> BCM4360

Dependant of AirPortBrcm4360.kext (IO80211Family.kext plugin)

* device-id pci12e4,4331 >> BCM94331
* device-id pci12e4,4353 >> BCM943224.

Many users including myself have used Fenvi T919 or Fenvi HB1200 PCI-express cards (Wi-Fi + Bluetooth combo) which have worked since at least High Sierra OOTB, without the need for additional drivers, automatically installed by macOS and recognized as Airdrop and Bluetooth by macOS.

The 2 Fenvi cards have the BCM4360 Wi-Fi chipset so they have stopped working in Sonoma. Bluetooth works well, as in Ventura and earlier. I have already commented that this is a serious inconvenience because the features associated with the Apple ecosystem are lost: Airdrop, Continuity, iPhone camera...

As extra information, Macs that can officially update to Sonoma have these Broadcom Wi-Fi:

* 2017 iMacPro1,1 / 2018/2019 MacBookAir8,x -> BCM4355 (pci14e4,43dc)
* 2018 MacMini8,1 / 2018/2019 MacBookPro15,x / 2019 iMac19,x / 2019 MacPro7,1 / 2019/2020 MacBookPro16,x / 2020 iMac20,x -> BCM4364 (pci14e4,4464)
* 2020 MacBookAir9.1 -> BCM4377b (pci14e4.4488).

They are chipsets soldered on the board that are not sold loose on the market and we cannot got them to install a Wi-Fi compatible with Sonoma in our Hack.

### Get back Fenvi Wi-Fi in Sonoma

OCLP developers have been working on this issue and have released a Sonoma-specific OCLP 0.6.9 beta that makes Wi-Fi work again like it did in Ventura. I know it is not the ideal situation, many of us want to have the system as similar as possible to a real Mac and OCLP has to apply root patches that force it to work by relaxing some macOS security rules. But what the OCLP team has achieved is a very big advance.

You have the instructions in this post (look for the **Hackintosh notes** section):

[Early preview of macOS Sonoma support now available!](https://github.com/dortania/OpenCore-Legacy-Patcher/pull/1077#issuecomment-1646934494)

To download the most recent version of this Sonoma-specific OCLP beta branch, look for the link with this text:

_Latest builds for the sonoma-development branch can be found below:_<br>
_Nightly.link: OpenCore-Patcher.app (Sonoma Development)_.

Note: OCLP developers prefer that we download OCLP beta from the link they post, for this reason I do not put a direct link here. It is a way to take users to the original post so that it can be read, which is highly recommended.

In summary, this is what to do:

* System Integrity Protection disabled: `csr-active-config=03080000`
* AMFI disabled: `boot-args = amfi=0x80`
* `Secure Boot Model = Disabled`
* Block com.apple.iokit.IOSkywalkFamily, setting MinKernel to 23.0.0 to ensure the patch is applied only in Sonoma
* Inject 3 extensions (Kexts folder and config.plist): IOSkywalk.kext, IO80211FamilyLegacy.kext and AirPortBrcmNIC.kext (IO80211FamilyLegacy.kext plugin) in this order, setting MinKernel to 23.0.0 to ensure they are injected only in Sonoma
* Reboot and apply OCLP root patch (Modern Wireless Network).

My Wi-Fi is Fenvi T919 so I have tried this pre-release version of OCLP 0.6.9. I have followed the instructions TO THE LETTER and they have worked well. I have Wi-Fi and Airdrop in Sonoma. Please note that _khronokernel_ instructions must be followed EXACTLY. In short, this version of OCLP 0.6.9 beta works, at least for me.
<br>
<p align="center">
<img width="640" src="Wifi active again.png">
</p>
<br>

Don't forget to enable (`Enabled=True`) the 3 added extensions and the blocked extension.
Important: com.apple.iokit.IOSkywalkFamily block must have `Enabled=True` and `Strategy=Exclude`. Otherwise, you may have kernel panic at boot.

Incremental updates are lost with this configuration, updates can be notified from Software Update but the full installation package is downloaded and not the delta package that only contains changes from the previous version. To obtain incremental updates you have to revert the OCLP root patch and restart but you lose Wi-Fi, keep this in mind if you depend on it to have Internet access, in this case do not revert root patch before proceeding with the update.

Note: After updating, **you must ALWAYS reapply root patch** since macOS overwrites the files modified by the patch, installing the original versions.

---

### AMFI and AMFIpass.kext

AMFI (Apple Mobile File Integrity) was originally seen on iOS but migrated to macOS in 10.12 Sierra, possibly in 2012 when GateKeeper and digitally signed code were introduced. In short, it is a technology that blocks the execution of non signed code. It consists of 2 components:

* `/usr/libexec/amfid` service run as root from '/System/Library/LaunchDaemons/com.apple.MobileFileIntegrity.plist`
* `/System/Library/Extensions/AppleMobileFileIntegrity.kext`.

AMFI must be enabled to grant third-party applications access to privacy-relevant services and/or peripherals, such as external cameras and microphones. But, with SIP and/or AMFI disabled (a necessary condition to apply OCLP root patches) the dialog box to grant access to those applications is not shown to the user so those peripherals simply cannot be used in applications like Zoom or MS Teams, for example.

AMFI is usually enabled but it has already been seen that OCLP root patches require disabling AMFI and SIP in order to be applied. To avoid the problem of peripherals not working with third-party applications, the OCLP team has developed the **AMFIPass.kext** extension that allows AMFI to be enabled when the system must operate with AMFI and SIP disabled, such as when using OCLP or applying root patches. This fixes the permissions issue and OCLP can apply the patches as if AMFI were disabled.

If macOS has previously given permissions to these third-party applications and then AMFI and/or SIP is disabled, these permissions are transferred and the new system maintains them. But in a clean installation they do not exist. This is the main problem that AMFIPass.kext tries to solve. Being able to root patch OCLP with AMFI enabled is just a positive side effect.

In summary, when applying OCLP root patches you can act in 2 different ways:

* with boot argument `amfi=0x80` without AMFIPass.kext. `amfi=0x80` is a bitmask that disables AMFI completely. The value 0x80 is equivalent to `AMFI_ALLOW_EVERYTHING`
* with AMFIPass.kext removing `amfi=0x80` and adding `-amfipassbeta` in boot args.

The `-amfipassbeta` boot argument is provided by AMFIPass.kext to override kernel version checking, so that the extension is loaded regardless of the macOS version. This way AMFIPass can work on macOS beta for which the extension does not yet have support.

I use AMFIPass.kext, removing `amfi=0x80`. If OCLP root patching fails due to this setting, I temporarily disable AMFI with the boot argument `amfi=0x80`, apply the patches, reboot, remove `amfi=0x80`, and reboot again.

(credits to [5T33Z0](https://github.com/5T33Z0) for much of the explanatory text on AMFI and AMFIPass.kext).
