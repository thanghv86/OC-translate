# AMD Radeon Performance Tweaks

## About
This chapter contains two approaches for improving the performance of AMD Radeon Graphics Cards when running macOS. The first method is pretty much standard to get the card running under macOS. The 2nd method has to be regarded as experimental. It makes use of modified SSDTs and a kext to improve the performance of AMD Radeon GPUs in OpenCL and Metal applications while also lowering the power consumption of the card. This method tries to mimic how the card would operate in a real Mac.

:warning: Use either method 1 or 2, not both!

## Method 1: For Navi GPUs (Recommended)
1. Add `SSDT-NAVI.aml` &rarr; Renames `PEGP` to `EGP0` so the GPU works (required for RX 5000/6000 Series Cards only). Also adds `HDAU` device for audio over HDMI.
2. Add the following Kexts to `EFI/OC/Kexts` and config.plist:
    - `Lilu.kext`
    - `Whatevergreen.kext`
3. Add Boot-arg `agdpmod=pikera` to config.plist → Fixes black screen issues on some Navi GPUs.

<details>
<summary>Contents of <strong>SSDT-NAVI.aml</strong> (click to reveal)</summary>

```swift
External (_SB_.PCI0, DeviceObj)
External (_SB_.PCI0.PEG0, DeviceObj)
External (_SB_.PCI0.PEG0.PEGP, DeviceObj)

Scope (\_SB)
{
    Scope (PCI0)
    {
        Scope (PEG0)
        {
            Scope (PEGP)
            {
                Method (_STA, 0, NotSerialized)  // _STA: Status
                {
                    If (_OSI ("Darwin"))
                    {
                        Return (Zero)
                    }
                    Else
                    {
                        Return (0x0F)
                    }
                }
            }

            Device (EGP0)
            {
                Name (_ADR, Zero)  // _ADR: Address
                Method (_STA, 0, NotSerialized)  // _STA: Status
                {
                    If (_OSI ("Darwin"))
                    {
                        Return (0x0F)
                    }
                    Else
                    {
                        Return (Zero)
                    }
                }

                Device (EGP1)
                {
                    Name (_ADR, Zero)  // _ADR: Address
                    Device (GFX0)
                    {
                        Name (_ADR, Zero)  // _ADR: Address
                        Name (_SUN, One)  // _SUN: Slot User Number
                        Method (_DSM, 4, NotSerialized)  // _DSM: Device-Specific Method
                        {
                            If ((Arg2 == Zero))
                            {
                                Return (Buffer (One)
                                {
                                     0x03                                             // .
                                })
                            }

                            Return (Package (0x02)
                            {
                                "hda-gfx", 
                                Buffer (0x0A)
                                {
                                    "onboard-2"
                                }
                            })
                        }
                    }

                    Device (HDAU)
                    {
                        Name (_ADR, One)  // _ADR: Address
                        Method (_DSM, 4, NotSerialized)  // _DSM: Device-Specific Method
                        {
                            If ((Arg2 == Zero))
                            {
                                Return (Buffer (One)
                                {
                                     0x03                                             // .
                                })
                            }

                            Return (Package (0x0A)
                            {
                                "AAPL,slot-name", 
                                "Built In", 
                                "device_type", 
                                Buffer (0x13)
                                {
                                    "Controller HDMI/DP"
                                }, 

                                "name", 
                                "High Definition Multimedia Interface", 
                                "model", 
                                Buffer (0x25)
                                {
                                    "High Definition Multimedia Interface"
                                }, 

                                "hda-gfx", 
                                Buffer (0x0A)
                                {
                                    "onboard-2"
                                }
                            })
                        }
                    }
                }
            }
        }
    }
}
```
</details>

## Method 2: Using AMD Radeon Patches by mattystonnie
> **Disclaimer**: Use at your own risk! In general, these patches have to be regarded as "experimental". They may work as intended but that's not guaranteed.

1. Select the SSDT corresponding to your GPU model located in the "mattystonnie" folder, export it as `.aml` and add it to add `EFI/OC/ACPI` and config.plist.
    - For **RX 580**: add `SSDT-RX580.aml` and `DTGP.aml`
    - For **RX 5500/5500XT**: add `SSDT-RX5500XT.aml` 
    - For **RX 5600/5700/5700XT**: add`SSDT-RX5700XT.aml`
    - For **RX Vega 64**: add `SSDT-RXVega64.aml`
2. Add the following Kexts to `EFI/OC/Kexts` and config.plist:
    - `DAGPM.kext` &rarr; Enables `AGPM` (Apple Graphics Power Management) Controller for AMD Cards which optimizes power consumption.
    - `Lilu.kext`
    - `Whatevergreen.kext`
3. Add Boot-arg `agdpmod=pikera` (for Navi GPUs only!) &rarr; Fixes black screen issues on some GPUs.
4. Add the following rename to `ACPI/Patch` (not required for Vega 56/64 and RX 580):
	```swift
	Find: 50454750
	Replace: 45475030
	Comment: Change PEGP to EGP0
	TableSignature: 44534454
	```
5. Save your config, reboot and run some benchmark tests for comparison.

### Addendum: SSDT vs. Device Properties

I've noticed that **SSDT-RX580** doesn't work as expected in macOS Catalina and beyond. In my tests, the performance didn't improve noticeably and power consumption wasn't optimized as well – around 100 Watts in idle which seems too high, imo. Also, `AGPMController` was present in IO Registry without `DAGPM.kext`, so this is not really a requirement either (same applies to `AGPMInjector.kext` by Pavo-IM).   

Looking for a solution, I've gone through the whole thread (90+ pages). On Page [35](https://www.tonymacx86.com/threads/amd-radeon-performance-enhanced-ssdt.296555/page-35#post-2114578) and following, I found another approach utilizing Device Properties to inject the data into macOS instead. This worked. It improves performance and reduces power consumption as well (70 Watts in idle instead of the previously used 100). 

I've added plists for both Clover and OpenCore to the "mattystonnie" folder. You can copy the included properties to the corresponding section of your config.plist. Ensure that the PCI paths and `AAPL,slot-name`[^1] match the ones used in your system and adjust them accordingly. Disable/delete the SSDTs and DAGPM.kext when using this method. 

[^1]: Follow this [guide](https://github.com/5T33Z0/OC-Little-Translated/tree/main/11_Graphics/GPU_Tab#3-obtaining-aaplslot-name-for-igpu-and-gpu) to to obtain the PCI path of a device and its `AAPL,slot-name` using Hackintool.

## Method 3: Injecting specific AMD Framebuffers via `DeviceProperties`

With this method, you don't need Whatevergreen and DRM works when using SMBIOS `iMac1,1` or `MacPro7,1`. 

:warning: Before attempting this, ensure you have a backup of your current EFI folder on a FAT32 formatted USB flash drive to boot from in case something goes wrong.

### Finding the PCIe path
- Run Hackintool
- Click on the "PCIe" Tab
- Find your GPU (should be listed as "Display Controller")
- Right-click the entry and select "Copy Device Path":</br>![PCIpath](https://user-images.githubusercontent.com/76865553/174430790-a35272cb-70fe-4756-a116-06c0f048e7a0.png)

### Config Edits
- Mount your EFI
- For NAVI Cards, add `SSDT-NAVI.aml` to `EFI/OC/ACPI` and the config.plist
- Disable Whatevergreen.kext
- Disable boot-arg `agdpmod=pikera`
- Under `DeviceProperties/Add` create a new child
- Set it to "Dictionary"
- Double Click its name and paste the PCI path:</br>![DevProps01](https://user-images.githubusercontent.com/76865553/174430804-b750e59a-46c7-4f38-aa0f-60977500b976.png)
- Double Click its name and paste the PCI path
- Create a new child
	- **Type**: String
	- **Name**: @0,name
	- **Value**: select the one for your GPU model from the list below:
		- **RX6900** &rarr; `ATY,Carswell`
		- **RX6800** &rarr; `ATY,Belknap`
 		- **RX6600/XT** &rarr; `ATY,Henbury`
		- **RX5700** &rarr; `ATY,Adder`
		- **RX5500** &rarr; `ATY,Python`
		- **RX570/580** &rarr; `ATY,Orinoco`
	- In this example, we use ATI,Henbury (without empty spaces):</br>![DevProps02](https://user-images.githubusercontent.com/76865553/174430822-f63c0cf0-c8a1-463f-901d-9053e8c7a981.png)
- Save and reboot.

**SOURCE**: [Insanelymac](https://www.insanelymac.com/forum/topic/351969-pre-release-macos-ventura/?do=findComment&comment=2786122)

## PowerPlay Table Generators for Radeon RX5700, Radeon VII and Vega 64

You can use PowerPlay Table Generators by MMChris to generate a `PP_PhmSoftPowerPlayTable` device property for [**Radeon VII**](https://www.insanelymac.com/forum/topic/340009-tool-radeon-vii-powerplay-table-generator-oc-uv-fan-curve/), [**Vega64**](https://www.hackintosh-forum.de/forum/thread/39923-tool-vega-64-powerplaytable-generator/) and [**RX5700**](https://www.insanelymac.com/forum/topic/340909-tool-amd-radeon-rx-5700-xt-powerplay-table-generator/) cards. These are basically Excel spreadsheets which generate the necessary hex values.

This way, you can inject all sorts of parameters into macOS to optimize the performance of your card such as: Power Limits, Clock Speeds, Fan Control and more without having to flash a modified vBIOS on your card.

## Credits & Resources
- **Files**:
	- Acidanthera for `Lilu.kext` and `WhateverGreen.kext`
	- mattystonnie for the SSDTs and original [**Guide**](https://www.tonymacx86.com/threads/amd-radeon-performance-enhanced-ssdt.296555/)
	- Baio1977 for `SSDT-NAVI.aml`
	- Toleda for `DAGPM.kext`
	- CMMMChris for PowerPlay Table Generators
	- [**Video Bitrate Test Files**](https://jell.yfish.us/) by Jellyfish
- **Additional Guides**:
	- [**Creating Custom PowerPlay tables and fan curves for Polaris Cards**](https://www.reddit.com/r/hackintosh/comments/hg56pv/guide_polaris_rx_560_580_etc_custom_powerplay/)
	- [**XFX RX 6600 XT in macOS Monterey**](https://github.com/perez987/rx6600xt-on-macos-monterey)