# Tool installation and basic use

## ESP-IDF

Official instructions - 
https://docs.espressif.com/projects/esp-idf/en/stable/esp32/get-started/linux-setup.html#installation-of-esp-idf-and-tools-on-linux

Install 
```
echo "deb [trusted=yes] https://dl.espressif.com/dl/eim/apt/ stable main" | sudo tee /etc/apt/sources.list.d/espressif.list
sudo apt update
sudo apt install eim-cli
eim install -i v6.0.1
```

Actvate
```
source ~/.espressif/tools/activate_idf_v6.0.1.sh
```

Dump flash
```
esptool read-flash 0x0 ALL esptool-dump.bin
```

Access UART terminal (Exit with `CTRL+]`)
```
idf.py monitor
```

## minicom

Use dmesg to find port

```
sudo apt install minicom
sudo dmesg
sudo minicom -D /dev/ttyUSB0 -b 115200
```

## Flashrom

Install
```
sudo apt install flashrom
```

Run
```
flashrom -p ch341a_spi -r flashrom-dump.bin
```

## Ghidra

Download
```
sudo apt install default-jdk
wget https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_12.1.2_build/ghidra_12.1.2_PUBLIC_20260605.zip
unzip ghidra_12.1.2_PUBLIC_20260605.zip
```

Run
```
cd ghidra_12.1.2_PUBLIC
./ghidraRun
```

Optionally install a Ghidra MCP (Model Context Protocol) to let AI tools interact with Ghidra
- https://github.com/bethington/ghidra-mcp

```
wget https://github.com/bethington/ghidra-mcp/releases/download/v5.14.1/GhidraMCP-5.14.1.zip
wget https://github.com/bethington/ghidra-mcp/releases/download/v5.14.1/bridge_mcp_ghidra.py
wget https://github.com/bethington/ghidra-mcp/releases/download/v5.14.1/requirements.txt
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
claude mcp add ghidra -- /home/you/ghidra-mcp/venv/bin/python /home/you/ghidra-mcp/bridge_mcp_ghidra.py
```

Update claude code permissions
```
{
  "permissions": {
    "allow": ["mcp__ghidra__*"]
  }
}
```

# Binary analysis

Get general image info
```
source ~/.espressif/tools/activate_idf_v6.0.1.sh
esptool --chip esp32c6 image-info dump.bin
```

## Extract partiton information

```
dd if=dump.bin bs=1 skip=$((0x8000)) count=3072 of=parttable.bin
python3 ~/.espressif/v6.0.1/esp-idf/components/partition_table/gen_esp32part.py parttable.bin partitions.csv
```

## Extract application partition

```
dd if=dump.bin of=factory_app.bin bs=1024 skip=64 count=8128
esptool --chip esp32c6 image-info factory_app.bin
```

Import to ghidra as RISC-V (32-bit, little endian)

## Memory map settings (derived from esptool image-info)
```
Segment 0 — DROM/IROM (mapped flash, code+rodata)

Block Name: irom0_text (or similar)
Start Address: 0x420b0020
Length: 0x1e8ac (end address 0x420ce8cb)
Block Type: File Bytes, source = your imported file
File Offset: 0x18
Permissions: Read + Execute (no Write — this is flash-mapped, read-only)

Segment 1 — DRAM/IRAM, byte-accessible (loaded RAM)

Block Name: iram0_text_a
Start Address: 0x40800000
Length: 0x1744 (end address 0x40801743)
Block Type: File Bytes
File Offset: 0x1e8cc
Permissions: Read + Write + Execute (this is real RAM, runtime-writable, and contains executable code)

Segment 2 — DROM/IROM (mapped flash, code+rodata)

Block Name: drom0_data (or irom0_text_2)
Start Address: 0x42000020
Length: 0xa5fec (end address 0x420a600b)
Block Type: File Bytes
File Offset: 0x20018
Permissions: Read + Execute

Segment 3 — DRAM/IRAM, byte-accessible (loaded RAM)

Block Name: iram0_text_b
Start Address: 0x40801744
Length: 0x11880 (end address 0x40812fc3)
Block Type: File Bytes
File Offset: 0xc600c
Permissions: Read + Write + Execute
```
