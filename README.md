# qcom-clean-boot

When you unlock the bootloader on Android devices, you're greeted by a bright-yellow warning


# Status

* **MediaTek**: Works. Tested.
* **Qualcomm**: In progress on the Galaxy Tab A9+ Wi-Fi (gta9pwifi, SM-X210). On Qualcomm SoC, splash assets are placed inside a UEFI firmware volume (`imagefv.elf`), making it less simple than dealing with MediaTek TAR-based partitions


## Content

```text
.
├── imagefv/                      # Extracted files from imagefv.elf
│   └── src/
│       ├── *.png                 # Screenshots used in this README
│       ├── custom_logo*.jpg.jpg  # Original bootloader warning images
│       └── orange_state*.jpg.jpg
├── custom_logo*.jpg.jpg          # Modified images (edited splash screens)
├── orange_state*.jpg.jpg
├── logo_gen_qcom.py              # Qualcomm splash generator (reference from [Codelinaro](https://git.codelinaro.org/clo/la/device/qcom/common/-/tree/qcom-devices.lnx.14.0.r12-rel/display/logo?ref_type=heads))
└── README.md                    
```

---

## MediaTek (`up_param.img`)

**Simple and confirmed**

| Step           | Command / Tool                                                | Notes                                                    |
| -------------- | ------------------------------------------------------------- | -------------------------------------------------------- |
| Dump partition | `dd if=/dev/block/by-name/up_param of=/sdcard/up_param.img`   | Verify the partition name           |
| Extract        | `tar xvf up_param.img`                                        | Images: `booting_warning.jpg`, `svb_orange.jpg` |
| Edit           | GIMP/Krita (fill with black or replace images)                | Check tolerance                     |
| Repack         | `tar -cvf up_param.tar *.jpg` → rename to `up_param.img` |                                                          |
| Flash back     | `dd if=/sdcard/up_param.img of=/dev/block/by-name/up_param`   |                                                          |
| Result       | Boot with no warning                                          |                                             |

---

## Qualcomm: (`imagefv.elf`)

On QC samsung devices, boot splash images are embedded differently:

```
imagefv.elf
├── [0x00000000-0x07] ELF header/stub
└── [0x00000008-EOF]  UEFI Firmware Volume (FV "3078") → Contains FREEFORM JPEG sections
```

**Limitation**:
7-Zip can *view* but not modify these files. UEFITool also doesn't support replacing these images due to parser limitations

**What's working & what's not**:

| Method                 | Result                                                                                             |
| ------------------------------- | -------------------------------------------------------------------------------------------------- |
| **7-Zip GUI**                   | Can view contents but refuses modification (`Not supported`)                                       |
| **UEFITool NE**                 | Option to `Replace body` is grayed out                                               |
| **Standard UEFITool v0.28**     | Displays contents but replacement still blocked (unknown FREEFORM section E0h) |
| **UEFIReplace / UEFIPatch CLI** | Promising, but missing `uefiextract` helper in Ubuntu package                                      |

---

## Qualcomm Splash Generation (`logo_gen_qcom.py`)

This script creates a Qualcomm `splash.img`, embedding the standard Qualcomm "SPLASH!!" header and dimensions.

```bash
python3 logo_gen_qcom.py my_logo.png # → splash.img
```

**Note:** Samsung Snapdragon devices don't have a dedicated `logo` partition, they store splash images in `imagefv.elf`. This script works for the devices with logo/splash partitions (like OnePlus)

---
 

| Issue                      | Why it's a problem         | 
| -------------------------- | -------------------------- |
| No `uefiextract` in Ubuntu | Can't extract FV easily    | 
| 7-Zip CLI limitation       | Can't modify FV            |
| Finding proper GUIDs      | Needed for proper patching |

---

## Reproduce

```bash
adb shell su
dd if=/dev/block/by-name/imagefv of=/sdcard/imagefv.elf  # Can also retrieve from BL firware archive
binwalk -e imagefv.elf                                   # Finds FV header at offset 0x8
7z x imagefv.elf                                         # Terminal refuses to extract (not supported)
mogrify -fill black -colorize 100% custom_logo*.jpg.jpg orange_state*.jpg.jpg # Modify with ImageMagick
UEFITool (Win32) displays images clearly but `Replace body` option disabled  # Stuck here
```

---

## Resources 

* **MediaTek `up_param` solutions**: 
* **Qualcomm ImageFV details**: 
* **7-Zip CLI/GUI limitations**: 
* **UEFITool parsing issues**: 
* **Qualcomm splash format info**: 
* **UEFI image parsing vulnerabilities (LogoFAIL)**: 


## References
1.  [https://www.reddit.com/r/androidroot/comments/mo5zsi/how\_to\_turn\_off\_bootloader\_screen\_on\_samsung/](https://www.reddit.com/r/androidroot/comments/mo5zsi/how_to_turn_off_bootloader_screen_on_samsung/)
2.  [https://github.com/a14xm-dev/mtk\_clean\_boot](https://github.com/a14xm-dev/mtk_clean_boot)
3.  [https://xdaforums.com/t/possible-way-to-change-boot-splash.4200911/](https://xdaforums.com/t/possible-way-to-change-boot-splash.4200911/)
4.  [https://superuser.com/questions/73381/7zip-add-operation-not-supported](https://superuser.com/questions/73381/7zip-add-operation-not-supported)
5.  [https://www.tenforums.com/software-apps/160615-7-zip-add-zip-doesnt-work.html](https://www.tenforums.com/software-apps/160615-7-zip-add-zip-doesnt-work.html)
6.  [https://github.com/LongSoft/UEFITool/issues/42](https://github.com/LongSoft/UEFITool/issues/42)
7.  [https://forums.mydigitallife.net/threads/uefitool-uefi-firmware-image-viewer-and-editor.48979/page-15](https://forums.mydigitallife.net/threads/uefitool-uefi-firmware-image-viewer-and-editor.48979/page-15)
8.  [https://www.binarly.io/blog/finding-logofail-the-dangers-of-image\_parsing-during-system-boot](https://www.google.com/search?q=https://www.binarly.io/blog/finding-logofail-the-dangers-of-image_parsing-during-system-boot)
9.  [https://xdaforums.com/t/guide-how-to-change-boot-logo-splash-screen-for-snapdragon-devices-splash\_img.3470473/page-2](https://www.google.com/search?q=https://xdaforums.com/t/guide-how-to-change-boot-logo-splash-screen-for-snapdragon-devices-splash_img.3470473/page-2)
10. [https://github.com/WeAreFairphone/splash-imgs/blob/master/logo\_gen.py](https://github.com/WeAreFairphone/splash-imgs/blob/master/logo_gen.py)
11. [https://github.com/MlgmXyysd/Magic-Splash-Wand](https://github.com/MlgmXyysd/Magic-Splash-Wand)
12. [https://xdaforums.com/t/guide-g780f-android-11-remove-bootloader-warning.4255707/](https://xdaforums.com/t/guide-g780f-android-11-remove-bootloader-warning.4255707/)
