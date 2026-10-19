# Microsoldering basics and tools

- https://www.youtube.com/@nanofixca

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
You can also download esptool standalone - https://github.com/espressif/esptool
```
esptool read-flash 0x0 ALL esptool-dump.bin
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

# Binary analysis

Get general image info
```
source ~/.espressif/tools/activate_idf_v6.0.1.sh
esptool --chip esp32c6 image-info dump.bin
```
