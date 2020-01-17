

This is the first of three mini-posts where I will write about some fun discoveries

- **1/3: Dissecting a ~~piñata~~Router**
- 2/3: [CVE-2019-14919, CVE-2019-14920. Inspecting a router's firmware and reversing binaries to achieve RCE](https://github.com/MarioBartolome/ForFun/blob/master/HW-Hacking/Firmware-Inspection.md)
- 3/3: [CVE-2019-14918. Stored XSS via DHCP injection](https://github.com/MarioBartolome/ForFun/blob/master/HW-Hacking/XSS-Injection-via-DHCP-requests.md)

# 0x01/0x03 : Dissecting a ~~piñata~~Router

So I lost the router's administration password and wanted to access it to check out on some configuration. I know, "just press the reset button"... but who's got the time? Besides, I would need to thread a pen through that tiny hole to press it. 

And, it's boring. Yeah, that too. 

So let's tear this thing apart and see if I can find the pass by dumping the firmware.

What you will need: 

- Router
- Screwdriver
- A BusPirate
- A cask where you could bath under the full moonlight
- A goat
- Chalk (to draw inverted pentagrams were needed)

## Components

A router is made out of many things. There's sugar, spice and everything nic... no, wait, hang on, that's for another project...

There's RAM memory as shown on this image: 

![RAM memory IC](https://user-images.githubusercontent.com/23175380/64636572-412cf800-d402-11e9-9355-6eacf4527eac.jpg)

Here's the chipset:

![Ralink Chipset IC](https://user-images.githubusercontent.com/23175380/64636689-818c7600-d402-11e9-8906-a5fd4576e710.jpg)

And this cute little thing, there's something quite interesting, a CMOS Serial Flash IC by Macronix:

![CMOS IC](https://user-images.githubusercontent.com/23175380/64636730-9f59db00-d402-11e9-9029-17b8acd3c787.jpg)

How do I know is a Macronix CMOS Serial Flash? well... You know... Pick one of:

- [x] I'm some kind of almighty warlock with some powerful powers (remember the goat and the chalk?)  
- [ ] I have checked out the printed text on the IC and found the data-sheet publicly available on the internet.

Your call.

![IC Text](https://user-images.githubusercontent.com/23175380/64925597-7450fc00-d7f3-11e9-83c7-0ee642db4f66.jpg)

As described on its [datasheet](https://www.macronix.com/Lists/Datasheet/Attachments/7370/MX25L6406E,%203V,%2064Mb,%20v1.9.pdf) that's a 64MB CMOS Flash memory. 

It seems to use a serial interface to make its reads and writes. And now is when the funny part begins. This little IC here, is the one storing the firmware. To dump the firmware it I'll need to connect to those little tiny pins. 

Just being careful to connect them correctly as I could fry up something. If smells like bacon it's OK, it's probably my breakfast. Burned electronic components smell more like... well, smells like shit, pain and desperation all together. 

I could just follow the wiring diagram but... I don't trust it cause I'm some kind of sick mind-twisted bastard and I would mess with the wirings just for the fun, so I'll check the coms with an oscilloscope. 

Wiring only the GND, RX and TX, and booting up the router, should present some traffic on the IC.

![IC Pin-out](https://user-images.githubusercontent.com/23175380/64925808-3bfeed00-d7f6-11e9-948c-002347b0c41c.png)

The communication is usually only alive on the boot up as the CPU loads the firmware into memory. Otherwise, accessing the IC to perform reads and writes over a Serial interface would be very slow.

![Oscilloscope approves](https://user-images.githubusercontent.com/23175380/64636753-aaad0680-d402-11e9-93ff-1660895a8956.jpg)

So everything seems fine. Let's wire this thing up!

## ~~Burning~~Wiring

There's many ways to do the wirings. One is soldering directly to the pins. Another would be to unsolder the IC from the board (this is actually the cleanest way), I could use a SOIC clip to attach to the pins, or some oscilloscope probes. 

I'll be using the last one as I happen to have some laying around. 

To wire it up, I followed the next table:

| CMOS  | BusPirate |
|-------|-----------|
| CS#   | CS        |
| SI    | MOSI      |
| SO    | MISO      |
| SCLK  | CLK       |
| WP#   | 3.3V      |
| HOLD# | 3.3V      |
| VCC   | 3.3V      |
| GND   | GND       |

It will look like a mess... But well, it's actually a mess
![Wiring mess](https://user-images.githubusercontent.com/23175380/64636803-c1ebf400-d402-11e9-9b94-082d0b421f4d.jpg)

## Firmware Dump

Once connected there's many ways to dump the firmware. I'll be using ```flashrom``` as it is able to "speak" with many USB-SPI interface bridges (BusPirate among them):

```flashrom -p buspirate_spi:dev=/dev/ttyUSB0,spispeed=1M -r router_firmware_cmos.bin```

Flashrom rulez and recognizes the CMOS memory as one of many Macronix flash chip. The one I'm ~~breaking~~ using is a **MX25L6406E**/MX25L6436E. Thus the command becomes:

```flashrom -p buspirate_spi:dev=/dev/ttyUSB0,spispeed=1M -r router_firmware_cmos.bin -c MX25L6406E/MX25L6436E``` and...

![Flashrom dumps the FW](https://user-images.githubusercontent.com/23175380/64639233-bb13b000-d407-11e9-83b3-63658904bad5.png)

It could take a few minutes to dump the FW.

## Firmware analysis 

Afterwards, making use of ```binwalk```, the binary will be analyzed to attempt to find some known structures.

![Binwalk finds some known structures](https://user-images.githubusercontent.com/23175380/64637208-9b7a8880-d403-11e9-8b94-30f1da1c4f9b.png)

There seems to exist compressed LZMA data at `0x50040`. By using the ```-e ``` flag ```binwalk``` can be instructed to extract the areas it would consider as known file types. 
Once extracted, it will create a folder containing the files.

Again, the binwalk extraction is attempted, and a new section beginning at @0x404000, and containing LZMA compressed data, is extracted.

![Binwalk finds some known structures... again](https://user-images.githubusercontent.com/23175380/64637226-a7664a80-d403-11e9-9a3c-791127ef15c3.png)


Finally, binwalking over the ```404000``` created file shows some interesting stuff: 

![Binwalk extracts the file-system](https://user-images.githubusercontent.com/23175380/64637279-bcdb7480-d403-11e9-9adb-2cccdf99a370.png)


Yay! That's the router's file-system. It can be accessed and inspected as if it was your own! 
![File-system](https://user-images.githubusercontent.com/23175380/64637318-cf55ae00-d403-11e9-9983-2f98dc722228.png)







