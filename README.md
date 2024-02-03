## macOS 14 Sonoma on Z390 Aorus Elite using OpenCore

<p align="center">
<img width="256" src="Sonoma icon.png">
</p>

### Preface

Sonoma required fewer changes to OpenCore and kexts than were necessary to install older systems such as Big Sur, which was a big challenge for developers. This time, minor changes allowed Sonoma to be installed almost immediately after the first beta version was released. Of course, there have been problems to work hard on, not all solved as of today. For now I will note the loss of Wi-Fi with Broadcom chipsets used in Mac models before 2017 and in Fenvi PCI-e cards, widely used in Hackintoshes. A fix has been provided by the OCLP team, read this: [Get back Fenvi T919 and other Broadcom Wi-Fi on macOS 14 Sonoma thanks to OLCP](https://github.com/perez987/fenvi-t919-back-on-macos-sonoma-by-oclp/blob/main/README.md).

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

* It is advisable to have macOS 13.4 Ventura or later
* System Settings >> Software Update >> Beta Updates >> click on the info icon >> Disabled
* Choose macOS Sonoma 14.0
* Or get the app from App Store.

To create the USB installation media so you can install Sonoma from scratch:

- Get the complete installation package from Apple's servers. I use the app [Download Full Installer ](https://github.com/perez987/DownloadFullInstaller)(original by _scriptingosx_), main window shows all versions available for download from Big Sur to Sonoma
- The package is downloaded as InstallAssistant-14.0-build.number.pkg, double click on the package to generate Install macOS Sonoma beta.app in the Applications folder
- Format a USB stick of at least 16Gb with GUID partition scheme and Mac OS Plus (journaled) format, name it (e.g. USB)
- Open Terminal and run this command
`sudo /Applications/Install\ macOS\ Sonoma.app/Contents/Resources/createinstallmedia --volume /Volumes/USB --no-interaction`

<br>
<p align="center">
<img width="512" src="DownloadFullInstaller.png">
</p>
<br>

At the end, you can reboot from the USB device and begin Sonoma installation.

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

* Systems where the OCLP root patch has not been applied or has been reverted:
	- iMac19,1 can get incremental updates
	- iMacPro1,1 and MacPro7,1 require `revpatch=sbvmm` in boot-args along with RestrictEvents.kext to get incremental updates, without this setting you get full-size updates
* All systems that have the OCLP root patch applied receive full-size updates.

In summary, using iMac19.1 without RestrictEvents.kext I get update notifications but the updates are full-size.

After the system is updated, RestrictEvents.kext and the boot argument can be disabled because they are not required for normal Sonoma operation.
