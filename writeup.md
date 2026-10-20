# ESP32-C6 Traffic Light: Finding the Engineer Key

**Tools:** `xxd`, `dd`, esptool (`pip install esptool`), Ghidra 11 or newer.

---

## Step 1: Find the app inside the flash dump

An ESP32 flash dump holds a bootloader, a partition table, and the app. The partition table is always at `0x8000`:

```
xxd -s 0x8000 -l 0x60 dump.bin
```

It lists `nvs`, `phy_init` and `factory`. The `factory` entry is the app, and it starts at `0x10000`. Cut it out:

```
dd if=dump.bin of=app.bin bs=4096 skip=16      # 16 * 4096 = 0x10000
```

## Step 2: Get the memory map

The app is not one flat blob. It is made of segments, and each one is loaded at a different address. esptool lists them:

```
esptool --chip esp32c6 image_info app.bin
```

```
Segment   Length   Load addr   File offs  Memory types
      0  0x1e8ac  0x420b0020  0x00000018  DROM, IROM
      1  0x01744  0x40800000  0x0001e8cc  DRAM, BYTE_ACCESSIBLE, IRAM
      2  0xa5fec  0x42000020  0x00020018  DROM, IROM
      3  0x11880  0x40801744  0x000c600c  DRAM, BYTE_ACCESSIBLE, IRAM
```

> **Gotcha:** "File offs" points at each segment's 8-byte header. The data starts **8 bytes later**.

So the values to give Ghidra are:

| Block name | Base address | File offset (in app.bin) | Length  | What it is      |
|------------|--------------|--------------------------|---------|-----------------|
| irom       | 0x42000020   | 0x20020                  | 0xa5fec | code in flash   |
| drom       | 0x420b0020   | 0x20                     | 0x1e8ac | strings / const |
| iram       | 0x40800000   | 0x1e8d4                  | 0x1744  | code in RAM     |
| dram       | 0x40801744   | 0xc6014                  | 0x11880 | data in RAM     |

## Step 3: Load it into Ghidra

1. Create a new project, then **File → Import File → app.bin**.
2. Set **Format** to `Raw Binary`. Set **Language** to `RISCV:LE:32:default`, because the ESP32-C6 is a 32-bit RISC-V chip.
3. Click **Options…** and fill in the `irom` row: Block Name `irom`, Base Address `0x42000020`, File Offset `0x20020`, Length `0xa5fec`.
4. Open the program. When Ghidra asks whether to analyze it, click **No** for now.
5. **File → Add To Program → app.bin**, then use **Options…** to add the `drom` row. Repeat for `iram` and `dram`.
6. **Analysis → Auto Analyze → Analyze**, keeping the default settings. This takes about a minute.

## Step 4: Find the code that checks the key

1. Open **Search → For Strings** and filter for `engineer`.
2. Double-click `Engineer mode authenticated`.
3. In the Listing, double-click the **XREF** next to the string. This jumps to the function that uses it (`FUN_ram_420140f2`).
4. The Decompiler window shows something like this:

```c
iVar1 = FUN_ram_4080956e(param_1, "Underscore write entry");
if (iVar1 == 0) {
    DAT_ram_4081b670 = 1;                       // "authenticated" flag
    log(..., "Engineer mode authenticated");
}
else {
    log(..., "Failed engineer authentication attempt");
}
```

`param_1` is the code you typed. It gets compared against a fixed value.

## Step 5: Confirm what the compare does

1. Double-click `FUN_ram_4080956e`. It walks both strings 4 bytes at a time using the `0x7f7f7f7f` mask trick and returns 0 when they match. That is newlib's standard **`strcmp`**. Press `L` to rename it.
2. The second argument comes from `lui a1,0x420b5 / addi a1,a1,-0x5e4`, which equals `0x420b4a1c`. Double-click it. It is a plain NUL-terminated string in read-only flash:

```
420b4a1c  55 6e 64 65 72 73 63 6f 72 65 20 77 72 69 74 65  Underscore write
420b4a2c  20 65 6e 74 72 79 00                             ' entry\0'
```

There is no hashing and no obfuscation. The key is the string itself.

3. (Optional) Right-click `DAT_ram_4081b670` and choose **References → Show References to**. The flag is set to 1 only here. The `control` command refuses to run unless the flag is set.

## Step 6: Use it

On the device's serial console, quote the code because it contains spaces:

```
traffic-mesh> engineer-mode "Underscore write entry"
Engineer mode enabled
```

---

**Key:** `Underscore write entry`

**Bonus:** devices that receive mesh commands never check the engineer flag. They only check that the target name matches their own. A hand-built UDP packet to `[ff02::1]:45678` can therefore change any light without the key.
