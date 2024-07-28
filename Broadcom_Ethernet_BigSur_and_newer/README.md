# Restore Broadcom Ethernet on Big Sur - Sequoia
Grab [**CatalinaBCM5701Ethernet.kext**](https://github.com/dortania/OpenCore-Legacy-Patcher/tree/main/payloads/Kexts/Ethernet) provided by OCLP.

#### How to use: 
1. Either add your device-id in the kext's Info.plist, **or** add these device properties:

|Key* | Value |Type |
|-|-|-|
| device-id | B4160000 | Data |
| compatible | pci14e4,16b4 | String |

2. Add the kext to your OC/Kexts folder, and make sure it is reflected in your config.plist.
3. Set MinKernel to **20.0.0**. <br>

<br>

Add these to **Kernel -> Patch** AS IT IS:
|Identifier*|Find|Replace|minKernel|Count| Comment |
|-|-|-|-|-|-|
|com.apple.iokit.CatalinaBCM5701Ethernet | E8CA9EFF FF668983 00050000 | B8B41600 00668983 00050000 | 20.0.0 | 1 | Broadcom BCM577XX Patch |


#### Cosmetic
This is optional. You could also apply a **Kernel -> Patch** to show the correct model in System Report, edit accordingly to match your model.
|Identifier*|Find|Replace|minKernel|Count| Comment |
|-|-|-|-|-|-|
|com.apple.iokit.CatalinaBCM5701Ethernet | 35373736 35 | 35373738 35 | 20.0.0 | 0 | SysReport 57765 -> 57785 (Cosmetic) |

3 <kbd>5</kbd> 3 <kbd>7</kbd> 3 <kbd>7</kbd> 3 <kbd>**6**</kbd> 3 <kbd>**5**</kbd> -> 3 <kbd>5</kbd> 3 <kbd>7</kbd> 3 <kbd>7</kbd> 3 <kbd>**8**</kbd> 3 <kbd>**5**</kbd>

You could use [HexFriend](http://hexfiend.com) to find and replace these values instead.

Credits: 
- **[Sunki](https://www.applelife.ru/threads/patching-applebcm5701ethernet-kext.27866/page-8#post-930901)** and **[Acidanthera](https://github.com/acidanthera/OpenCorePkg/blob/cb591b7671215b31dc4a2bc5b1e9da9c92eaebf4/Docs/Sample.plist#L837)** for the source of patch
- **[Andrey1970AppleLife](https://www.applelife.ru/threads/patching-applebcm5701ethernet-kext.27866/page-9#post-1031837)** for [guide](https://www.applelife.ru/threads/patching-applebcm5701ethernet-kext.27866/page-7#post-869709) and cosmetic patch I used in this guide
- Dortania/Khronokernel for patched CatalinaBCM5701Ethernet
