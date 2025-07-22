# QCOM-Clean-Boot

When you unlock the bootloader on Android devices, you're greeted by an annoying, bright-yellow warning:

> ⚠️ **"Your device software can't be trusted..."**


## Current Status

* **MediaTek**: Works. Tested on the Galaxy A14 5G (MT6833)
* **Qualcomm**: In progress on the Galaxy Tab A9+ Wi-Fi (gta9pwifi, SM-X210). On Qualcomm SoC, splash assets are hidden inside a UEFI firmware volume (`imagefv.elf`), making it trickier than MediaTek’s simpler TAR-based partitions.


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
└── README.md                     # This file
```

---

## MediaTek (`up_param.img`)

**Simple and confirmed working.**

| Step           | Command / Tool                                                | Notes                                                    |
| -------------- | ------------------------------------------------------------- | -------------------------------------------------------- |
| Dump partition | `dd if=/dev/block/by-name/up_param of=/sdcard/up_param.img`   | Verify the partition name first (\[Reddit]\[1])          |
| Extract        | `tar xvf up_param.img`                                        | Images revealed: `booting_warning.jpg`, `svb_orange.jpg` |
| Edit           | GIMP/Krita (fill with black or replace images)                | Check tolerance tips (\[GitHub]\[2])                     |
| Repack         | `tar -cvf up_param.tar *.jpg` → rename back to `up_param.img` |                                                          |
| Flash back     | `dd if=/sdcard/up_param.img of=/dev/block/by-name/up_param`   |                                                          |
| Result 🎉      | Boot with no warning                                          | Confirmed on A14 5G                                      |

---

## Qualcomm: (`imagefv.elf`)

On Qualcomm devices, boot splash images are embedded differently:

```
imagefv.elf
├── [0x00000000-0x07] ELF header/stub
└── [0x00000008-EOF]  UEFI Firmware Volume (FV "3078") → Contains FREEFORM JPEG sections
```

**Limitation**:
7-Zip can *view* but not modify these files. UEFITool also doesn't currently support replacing these images due to parser limitations

**What's working & what's not**:

| Method Attempted                | Result                                                                                             |
| ------------------------------- | -------------------------------------------------------------------------------------------------- |
| **7-Zip GUI**                   | Can view contents but refuses modification (`Not supported`)                                       |
| **UEFITool NE**                 | Option to `Replace body` is grayed out (\[Issue]\[7])                                              |
| **Standard UEFITool v0.28**     | Displays contents but replacement still blocked (unknown FREEFORM section E0h) (\[MDL Forums]\[8]) |
| **UEFIReplace / UEFIPatch CLI** | Promising, but missing `uefiextract` helper in Ubuntu package                                      |

---

## Qualcomm Splash Generation (`logo_gen_qcom.py`)

This script creates a classic Qualcomm `splash.img`, embedding the standard Qualcomm "SPLASH!!" header and dimensions, optionally using RLE compression.

```bash
python3 logo_gen_qcom.py my_logo.png # → splash.img
```

**Note:** Samsung Snapdragon devices don't have a dedicated `logo` partition. Instead, they store splash images within `imagefv.elf`. This script works for the devices with logo/splash partitions (like OnePlus).

---

## Roadblocks 

| Issue                      | Why it's a problem         | Possible solution(s)                                              |
| -------------------------- | -------------------------- | ----------------------------------------------------------------- |
| No `uefiextract` in Ubuntu | Can't extract FV easily    | Build from source or switch to \[Magic-Splash-Wand]\[12] (Python) |
| 7-Zip CLI limitation       | Can't modify FV            | Stick to specialized UEFI tools (UEFITool CLI/GUI)                |
| Finding Correct GUIDs      | Needed for proper patching | Use `strings` command or deep-grep the extracted content          |

---

## Reproducing Current Issue (Qualcomm Path)

```bash
adb shell su
dd if=/dev/block/by-name/imagefv of=/sdcard/imagefv.elf  # ✔ Dump works fine
binwalk -e imagefv.elf                                   # ✔ Finds FV header at offset 0x8
7z x imagefv.elf                                         # ❌ CLI refuses to extract (not supported)
mogrify -fill black -colorize 100% custom_logo*.jpg.jpg orange_state*.jpg.jpg # ✔ Images modified with ImageMagick
UEFITool (Win32) displays images clearly but `Replace body` option disabled  # 🚫 Stuck here
```

---

## Resources & Discussions

* **MediaTek `up_param` solutions**: \[Reddit]\[1], \[XDA]\[13]
* **Qualcomm ImageFV details**: \[XDA Forums]\[3]
* **7-Zip CLI/GUI limitations**: \[SuperUser]\[5], \[TenForums]\[6]
* **UEFITool parsing issues**: \[GitHub Issue #42]\[7], \[MyDigitalLife Forums]\[8]
* **Qualcomm splash format info**: \[XDA Guide]\[10], \[Reference Python script]\[11]
* **UEFI image parsing vulnerabilities (LogoFAIL)**: \[Binarly]\[9]


---

**Footnotes:** 
\[1]: [https://www.reddit.com/r/androidroot/comments/mo5zsi/how\_to\_turn\_off\_bootloader\_screen\_on\_samsung/](https://www.reddit.com/r/androidroot/comments/mo5zsi/how_to_turn_off_bootloader_screen_on_samsung/)
\[2]: [https://github.com/a14xm-dev/mtk\_clean\_boot](https://github.com/a14xm-dev/mtk_clean_boot)
\[3]: [https://xdaforums.com/t/possible-way-to-change-boot-splash.4200911/](https://xdaforums.com/t/possible-way-to-change-boot-splash.4200911/)
\[5]: [https://superuser.com/questions/73381/7zip-add-operation-not-supported](https://superuser.com/questions/73381/7zip-add-operation-not-supported)
\[6]: [https://www.tenforums.com/software-apps/160615-7-zip-add-zip-doesnt-work.html](https://www.tenforums.com/software-apps/160615-7-zip-add-zip-doesnt-work.html)
\[7]: [https://github.com/LongSoft/UEFITool/issues/42](https://github.com/LongSoft/UEFITool/issues/42)
\[8]: [https://forums.mydigitallife.net/threads/uefitool-uefi-firmware-image-viewer-and-editor.48979/page-15](https://forums.mydigitallife.net/threads/uefitool-uefi-firmware-image-viewer-and-editor.48979/page-15)
\[9]: [https://www.binarly.io/blog/finding-logofail-the-dangers-of-image-parsing-during-system-boot](https://www.binarly.io/blog/finding-logofail-the-dangers-of-image-parsing-during-system-boot)
\[10]: [https://xdaforums.com/t/guide-how-to-change-boot-logo-splash-screen-for-snapdragon-devices-splash-img.3470473/page-2](https://xdaforums.com/t/guide-how-to-change-boot-logo-splash-screen-for-snapdragon-devices-splash-img.3470473/page-2)
\[11]: [https://github.com/WeAreFairphone/splash-imgs/blob/master/logo\_gen.py](https://github.com/WeAreFairphone/splash-imgs/blob/master/logo_gen.py)
\[12]: [https://github.com/MlgmXyysd/Magic-Splash-Wand](https://github.com/MlgmXyysd/Magic-Splash-Wand)
\[13]: [https://xdaforums.com/t/guide-g780f-android-11-remove-bootloader-warning.4255707/](https://xdaforums.com/t/guide-g780f-android-11-remove-bootloader-warning.4255707/)
