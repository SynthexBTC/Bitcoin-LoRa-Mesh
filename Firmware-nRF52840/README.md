# Flashing MeshCore on ProMicro nRF52

## 1. Get the right file

1. Open the page for the role you want:
   - Companion : https://flasher.meshcore.io/promicro-nrf52-faketec/companion-bluetooth/  
     <img src="images/Meshcore-Download-Firmware-BLE.png">
   - Repeater  : https://flasher.meshcore.io/promicro-nrf52-faketec/repeater/  
     <img src="images/Meshcore-Download-Firmware-RPT.png">
2. Click **Download** to get a copy of the firmware files.
3. Pick the `.uf2` file

## 2. Enter bootloader mode

1. Plug the board into your computer with a **data-capable** USB cable.
2. **Press the RESET button TWICE** in quick succession.
3. The board should appear as a USB drive (drive name: PROMICRO).  
   <img src="images/Meshcore-Firmware-Folder.png">

## 3. Flash

1. Drag and drop the `.uf2` file onto the USB drive.  
2. Wait for the copy to finish. The drive should disappear and the board should reboot.
     
   Tips : Windows copy error 0x80070022 ("The drive does not contain the right disk"), shown right after dropping the .uf2 file.  
   The board reboots as soon as the file is written, so the drive vanishes before Windows confirms the copy.  
   This kind of error is normal and safe to ignore.  
