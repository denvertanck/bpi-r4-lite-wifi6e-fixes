# Banana Pi BPI-R4 LITE: Wi-Fi 6E (MT7916) OpenWrt Hardware Patches

> **⚠️ CRITICAL ARCHITECTURAL WARNING**
> These patches are mathematically mapped and engineered **strictly for the Banana Pi BPI-R4 LITE**. 
> Do not apply these configurations to the standard BPI-R4 main board. The GPIO pin layouts and PCIe lane multiplexing are physically different. Applying this to a standard BPI-R4 may result in kernel panics or hardware instability. BUT IF you ever try the fix on a BPI-R4 please share the result with me. 

### The Firmware Baseline: OpenWrt 25.12.4
**Critical Context:** These patches were mapped, compiled, and mathematically validated against **OpenWrt version 25.12.4**. As the OpenWrt ecosystem evolves, upstream developers may permanently integrate the PCIe multiplexing fix for dual-mPCIe Banana Pi architectures into future releases. If you are compiling a version strictly newer than 25.12.4, verify the upstream target commits before applying the DTS patch to prevent structural collisions.

---

## The Mechanical Diagnosis
When compiling OpenWrt from source for the BPI-R4 Lite paired with a Wi-Fi 6E module (specifically utilizing the MT7916 chipset, such as the AsiaRF AW7916-NPD), two distinct physical and driver-level failures occur:

1. **The PCIe Multiplexing Failure:** The default Device Tree Source (DTS) for the Lite board does not properly activate the secondary PCIe interface (`pcie1`) or trigger the correct GPIO switch required to route the data lanes to the Wi-Fi card.
2. **The 6GHz Spectrum Drop:** The MediaTek MT76 MAC80211 driver (`eeprom.c`) contains a fallback logic that frequently fails to lock the MT7916 module into the 6GHz spectrum, aggressively downgrading the radio broadcast to 5GHz.

This repository contains the two mathematical patch files required to bypass these structural limitations.

---

## The Patch Architecture

### 1. The PCIe Lane Fix (`0001-bpi-r4-lite-pcie-fix.patch`)
**Target:** `target/linux/mediatek/dts/mt7987a-bananapi-bpi-r4-lite.dts`

This Unified Diff patch injects the necessary hardware triggers directly into the compiled Device Tree. 
* It explicitly declares the `&pcie1` node status as `okay`.
* It binds a GPIO hog to PIN 11 (`aw35710qnr_sel`), setting it to `GPIO_ACTIVE_HIGH` and `output-low`. This physical voltage switch mechanically forces the board to route the PCIe lanes to the M.2 Wi-Fi slot.

### 2. The MT7916 6GHz EEPROM Lock (`999-absolute-mt7916-6ghz.patch`)
**Target:** `package/kernel/mac80211/patches/build/` (Modifies `drivers/net/wireless/mediatek/mt76/mt7915/eeprom.c`)

This patch overrides the default EEPROM parsing logic in the Linux kernel. 
* When the driver detects `MT_EE_V2_BAND_SEL_6GHZ` or `MT_EE_V2_BAND_SEL_5GHZ_6GHZ`, this code forcibly sets `phy->mt76->cap.has_6ghz = true;` and mathematically strips the 5GHz fallback loop.
* It ensures the MT7916 card remains permanently locked to the 6GHz frequency band.

---

## The Execution Protocol (How to Apply)

To implement these fixes into your OpenWrt build environment, execute the following matrix from the root directory of your OpenWrt source tree.

**Step 1: Download the Patches**
```bash
wget [https://raw.githubusercontent.com/denvertanck/bpi-r4-lite-wifi6e-fixes/main/patches/0001-bpi-r4-lite-pcie-fix.patch](https://raw.githubusercontent.com/denvertanck/bpi-r4-lite-wifi6e-fixes/main/patches/0001-bpi-r4-lite-pcie-fix.patch)
wget [https://raw.githubusercontent.com/denvertanck/bpi-r4-lite-wifi6e-fixes/main/patches/999-absolute-mt7916-6ghz.patch](https://raw.githubusercontent.com/denvertanck/bpi-r4-lite-wifi6e-fixes/main/patches/999-absolute-mt7916-6ghz.patch)

```

**Step 2: Inject the DTS PCIe Fix**

```bash
patch -p1 < 0001-bpi-r4-lite-pcie-fix.patch

```

**Step 3: Inject the MAC80211 EEPROM Fix**
Move the kernel patch into the specific build directory so the OpenWrt engine automatically applies it during the MT76 compilation phase.

```bash
mkdir -p package/kernel/mac80211/patches/build/
mv 999-absolute-mt7916-6ghz.patch package/kernel/mac80211/patches/build/

```

**Step 4: Execute the Build**
Clear the cached MAC80211 package to ensure the new C code is processed, then compile the firmware.

```bash
make package/kernel/mac80211/clean
make -j$(nproc) V=s

```

After successfully flashing the resulting `sysupgrade.itb` file to your BPI-R4 Lite, your AW7916 Wi-Fi 6E module will be structurally recognized by the PCIe bus and broadcast natively on the 6GHz spectrum.
