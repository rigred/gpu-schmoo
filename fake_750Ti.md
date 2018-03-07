# GTX 750Ti
## About card
The card claimes to be Nvidia GTX 750Ti with 4GB of DDR5 (?! at the time of writing not yet released).
The card in Linux shows up as GM107 (1st gen Maxwell) but with GF116 High Definition Audio controller (aka GTX550Ti).

![fake listing](fake750ti.png "lspci with fake BIOS")

It  It has 8*2Gb (both sides of PCB) SK Hynix [H5GQ2H24AFR](https://www.skhynix.com/eolproducts.view.do?pronm=GDDR5+SDRAM&srnm=H5GQ2H24AFR&rk=26&rc=graphics "Info from manufacturer") but after flash has only 768MB GDDR5. Works with 550Ti.rom BIOS from [linustechtips forum](https://linustechtips.com/main/topic/880010-cheap-fake-ebay-gpu-thread/ "Link to topic").

![fake listing](flashed.png "lspci")

## Flashing
Necessary hardware:
* CH341A USB MinProgrammer

![Black CH341A USB dongle](CH341A.jpg "Black CH341A USB dongle")
* SOP8 flash test clips with SOP16/8-DIP8 adapter

![clips and adapter](clips-adapter.jpg "Testing clips with adapter")
Necessary software:
* [Ch341Prog](https://github.com/setarcos/ch341prog)
* [GTX 550Ti ROM](./550ti.rom)
1. Clone Ch341Prog"
```
git clone https://github.com/setarcos/ch341prog
```
2. Compile it:
```
make
```
3. Make it executable
```
chmod +x ch341prog
```
4. Unscrew 4 screws holding heatsink on bottom of PCB, unplug fan.
5. Connect the clip to FM25F02A chip. Pin one is markd by ⋅⋅* on board, red wire on clips and by number one on adapter. Unlock the USB programmer and put adapter in socket so numbers 4 and 5 are closer to USB connector.
6. Plug programmer to USB and execute ch341prog to see info:
```
sudo ./ch341prog -i
```
7. Backup fake BIOS:
```
sudo ./ch341prog -r fake750ti.rom
```
8. Erase chip:
```
sudo ./ch341prog -e
```
8. Flash new BIOS:
```
sudo ./ch341prog -w 550ti.rom
```
9. You will get error when reading back but you can dump what you just written and compare it with original file. I should be same.
10. Put it all together, apply thermal paste, test it and have fun.
