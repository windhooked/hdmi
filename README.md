# hdmi

[![Build Status](https://travis-ci.com/hdl-util/hdmi.svg?branch=master)](https://travis-ci.com/hdl-util/hdmi)

SystemVerilog code for HDMI 1.4a video/audio output on an [FPGA](https://simple.wikipedia.org/wiki/Field-programmable_gate_array).

## Why?

Most free and open source HDMI source (computer/gaming console) implementations actually output a DVI signal, which HDMI sinks (TVs/monitors) are backwards compatible with. To support audio and other HDMI-only functionality, a true HDMI signal must be sent. The code in this repository lets you do that without having to license an HDMI IP block from anyone.

### Demo: VGA-compatible text mode, 720x480p on a Dell Ultrasharp 1080p Monitor

![GIF showing VGA-compatible text mode on a monitor](demo.gif)

## Usage

1. Take files from `src/` and add them to your own project. If you use [hdlmake](https://hdlmake.readthedocs.io/en/master/), you can add this repository itself as a remote module.
1. Other helpful modules for displaying text / generating sound are also available in this GitHub organization.
1. Consult the simple usage example in `top/top.sv`.
1. See [hdmi-demo](https://github.com/hdl-util/hdmi-demo) for code that runs the demo as seen the demo GIF.
1. Read through the parameters in `hdmi.sv` and tailor any instantiations to your situation.
1. Please create an issue if you run into a problem or have any questions. Make sure you have consulted the troubleshooting section first.

### Platform Support

- [x] Altera
- [ ] Xilinx (untested but should work)
- [ ] Lattice (unknown)

### To-do List (upon request)
- [x] 24-bit color
- [x] Data island packets
	- [x] Null packet
	- [x] ECC with BCH systematic encoding GF(2^8)
	- [x] Audio clock regeneration
	- [x] L-PCM audio
		- [x] 2-channel
		- [ ] 3-channel to 8-channel
	- [ ] 1-bit audio
	- [x] Audio InfoFrame
	- [x] Auxiliary Video Information InfoFrame
	- [x] Source Product Descriptor InfoFrame
	- [ ] MPEG Source InfoFrame
		- NOTE—Problems with the MPEG Source Infoframe have been identified that were not able to be fixed in time for CEA-861-D. Implementation is strongly discouraged until a future revision fixes the problems
	- [ ] Gamut Metadata
- [x] Video formats 1, 2, 3, 4, 16, 17, 18, and 19
- [x] VGA-compatible text mode
	- [x] IBM 8x16 font
	- [ ] Alternate fonts
- [ ] Other color formats (YCbCr, deep color, etc.)
- [ ] Support other video id codes
	- [ ] Interlaced video
	- [ ] Pixel repetition
- [ ] Special I/O features
	- [x] Double Data Rate I/O (DDRIO)


### Pixel Clock

You'll need to set up a PLL for producing the two HDMI clocks. The pixel clock for each supported format is shown below:

|Video Resolution|Video ID Code(s)|Refresh Rate|Pixel Clock Frequency|
|---|---|---|---|
|640x480|1|60Hz|25.2MHz|
|640x480|1|59.94Hz|25.175MHz|
|720x480|2, 3|60Hz|27.027MHz|
|720x480|2, 3|59.94Hz|27MHz|
|1280x720|4|60Hz|74.25MHz|
|1280x720|4|59.94Hz|74.176MHz|
|1920x1080|16|60Hz|148.5MHz|
|1920x1080|16|59.94Hz|148.352MHz|
|720x576|17, 18|50Hz|27MHz|
|1280x720|19|50Hz|74.25MHz|
|3840x2160|97, 107|60Hz|594MHz|

The second clock is a clock 10 times as fast as the pixel clock. Even if your FPGA only has a single PLL, the Altera MegaWizard (or the Xilinx equivalent) should still be able to produce both. You can avoid using two different multiplication factors, with the DDRIO feature, which only requires the second clock to be 5 times as fast.

### L-PCM Audio Bitrate / Sampling Frequency

Both audio bitrate and frequency are specified as parameters of the HDMI module. Bitrate can be any value from 16 through 24. Below is a simple mapping of sample frequency to the appropriate parameter

**WARNING: the audio can be REALLY LOUD if you use the full dynamic range with hand-generated waveforms! Using less dynamic range means you won't be deafened! (i.e. audio_sample >> 8 )**

|Sampling Frequency|AUDIO_RATE value|
|---|---|
|32 kHz|32000|
|44.1 kHz|44100|
|88.2 kHz|88200|
|176.4 kHz|176400|
|48 kHz|48000|
|96 kHz|96000|
|192 kHz|192000|


### Source Device Information Code

This code is sent in the Source Product Description InfoFrame via `SOURCE_DEVICE_INFORMATION` to give HDMI sinks an idea of what capabilities an HDMI source might have. It may be used for displaying a relevant icon in an input list (i.e. DVD logo for a DVD player).

|Code|Source Device Information|
|---|---|
|0x00|Unknown|
|0x01|Digital Set-top Box|
|0x02|DVD Player|
|0x03|Digital VHS|
|0x04|HDD Videorecorder|
|0x05|Digital Video Camera|
|0x06|Digital Still Camera|
|0x07|Video CD|
|0x08|Game|
|0x09|PC General|
|0x0a|Blu-Ray Disc|
|0x0b|Super Audio CD|
|0x0c|HD DVD|
|0x0d|Portable Media Player|

### Things to be aware of / Troubleshooting

* Limited resolution: some FPGAs don't support I/O at speeds high enough to achieve 720p/1080p
    * Workaround: use DDR/other special I/O features like I/O serializers
	* Workaround: Altera FPGA users can try to specify speed grade C6 and see if it works, though yours may be C7 or C8. If it doesn't work, try enabling DDRIO.
* FPGA does not support TMDS: many FPGAs without a dedicated HDMI output don't support TMDS
    * You should be able to directly use LVDS (3.3v) instead, tested up to 720x480
    * This might not work if your video has a high number of transitions or you plan to use higher resolutions
    * Solution: AC-couple the 3.3v LVDS wires to by adding 100nF capacitors in series, as close to the transmitter as possible
        * Why? TMDS is current mode logic, and driving a CML receiver with LVDS is detailed in [Figure 9 of Interfacing LVDS with other differential-I/O types](https://m.eet.com/media/1135468/330072.pdf)
            * Resistors are not needed since Vcc = 3.3v for both the transmitter and receiver
        * Example: See `J13`, on the [Arduino MKR Vivado 4000 schematic](https://content.arduino.cc/assets/vidor_c10_sch.zip), where LVDS IO Standard pins on a Cyclone 10 FPGA have 100nF series capacitors
* Poor wiring: if you're using a breakout board or long lengths of untwisted wire, there might be a few pixels that jitter due to interference
    * Make sure you have all the necessary pins connected (GND pins, etc.)
    * Try switching your HDMI cable; some cheap cables like [these I got from Amazon](https://www.amazon.com/gp/product/B01JO9PB7E/) have poor shielding
* Hot-Plug unaware: all modules are unaware of hotplug
    * This shouldn't affect anything in the long term; the only stateful value is `hdmi.tmds_channel[2:0].acc`
    * You should decide hotplug behavior (i.e. pause/resume on disconnect/connect, or ignore it)
* EDID not implemented: it is assumed you know what format you want at synthesis time, so there is no dynamic decision on video format
    * To be implemented in a display protocol independent manner
* SCL/SCA voltage level: though unused by this implementation...it is I2C on a 5V logic level, as confirmed in the [TPD12S016 datasheet](https://www.ti.com/lit/ds/symlink/tpd12s016.pdf), which is unsupported by most FPGAs
    * Solution: use a bidirectional logic level shifter compatible with I2C to convert 3.3v LVTTL to 5v
    * Solution: use 3.3-V LVTTL I/O standard with 6.65k pull-up resistors to 3.3v (as done in `J13` on the [Arduino MKR Vivado 4000 schematic](https://content.arduino.cc/assets/vidor_c10_sch.zip))
	* Emailed Arduino support: safe to use as long as the HDMI slave does not have pull-ups

## Licensing

Dual-licensed under Apache License 2.0 and MIT License.

### HDMI Adoption

I am NOT a lawyer, the below advice is given based on discussion from [a Hacker News post](https://news.ycombinator.com/item?id=22279308) and my research.

HDMI itself is not a royalty free technology, unfortunately. You are free to use it for testing, development, etc. but to receive the HDMI LA's (licensing administration) blessing to create and sell end-user products:


> The manufacturer of the finished end-user product MUST be a licensed HDMI Adopter, and
> The finished end-user product MUST satisfy all requirements as defined in the Adopter Agreement including but not limited to passing compliance testing either at an HDMI ATC or through self-testing.


Becoming an adopter means you have to pay a flat annual fee (~ $1k-$2k) and a per device royalty (~ $0.05). If you are selling an end-user device and DO NOT want to become an adopter, you can turn on the `DVI_OUTPUT` parameter, which will disable any HDMI-only logic, like audio.

Please consult your lawyer if you have any concerns. Here are a few noteworthy cases that may help you make a decision:

* Arduino LLC is not an adopter, yet sells the [Arduino MKR Vidor 4000](https://store.arduino.cc/usa/mkr-vidor-4000) FPGA 
    * It has a micro-HDMI connector
    * [Having an HDMI connector does not require a license](https://electronics.stackexchange.com/questions/28202/legality-of-using-hdmi-connectors-in-non-hdmi-product)
    * Official examples provided by Arduino on GitHub only perform DVI output
    * It is a user's choice to program the FPGA for HDMI output
    * Therefore: the device isn't an end-user product under the purview of HDMI LA
* Unlicensed DisplayPort to HDMI cables (2011)
    * [Articles suggests that the HDMI LA can recall illegal products](https://www.pcmag.com/archive/displayport-to-hdmi-cables-illegal-could-be-recalled-266671?amp=1).
    * But these cables [are still sold on Amazon](https://www.amazon.com/s?k=hdmi+to+displayport+cable)
    * Therefore: the power of HDMI LA to enforce licensing is unclear
* [Terminated Adopters](https://hdmi.org/adopter/terminated)
    * There are currently 1,043 terminated adopters
    * Includes noteworthy companies like Xilinx, Lattice Semiconductor, Cypress Semiconductor, EVGA (!), etc.
    * No conclusion
* Raspberry Pi Trading Ltd is licensed
    * They include the HDMI logo for products
    * Therefore: Raspberry Pi products are legal, licensed end-user products

## Alternative Implementations

- [HDMI Intel FPGA IP Core](https://www.intel.com/content/www/us/en/programmable/products/intellectual-property/ip/interface-protocols/m-alt-hdmi-megacore.html): Stratix/Arria/Cyclone
- [Xilinx HDMI solutions](https://www.xilinx.com/products/intellectual-property/hdmi.html#overview): Virtex/Kintex/Zynq/Artix
- [Artix 7 HDMI Processing](https://github.com/hamsternz/Artix-7-HDMI-processing): VHDL, decode & encode
- [SimpleVOut](https://github.com/cliffordwolf/SimpleVOut): many formats, no auxiliary data

If you know of another good alternative, open an issue and it will be added.

## Reference Documents

*These documents are not hosted here! They are available on Library Genesis and at other locations.*

* [HDMI Specification v1.4a](https://libgen.is/book/index.php?md5=28FFF92120C7A2C88F91727004DA71ED)
* [HDMI Specification v2.0](https://b-ok.cc/book/5464885/1f0b4c)
* [EIA-CEA861-D.pdf](https://libgen.is/book/index.php?md5=CEE424CA0F098096B6B4EC32C32F80AA)
* [CTA-861-G.pdf](https://b-ok.cc/book/5463292/52859e)
* [DVI Specification v1.0](https://www.cs.unc.edu/~stc/FAQs/Video/dvi_spec-V1_0.pdf)
* [IEC 60958-1](https://ia803003.us.archive.org/30/items/gov.in.is.iec.60958.1.2004/is.iec.60958.1.2004.pdf)
* [IEC 60958-3](https://ia800905.us.archive.org/22/items/gov.in.is.iec.60958.3.2003/is.iec.60958.3.2003.pdf)
* [E-DDC v1.2](https://glenwing.github.io/docs/)

## Special Thanks

* Mike Field's (@hamsternz) demos of DVI and HDMI output for helping me better understand HDMI
	* http://www.hamsterworks.co.nz/mediawiki/index.php/Dvid_test
	* http://www.hamsterworks.co.nz/mediawiki/index.php/Minimal_DVI-D
* Jean P. Nicolle (fpga4fun.com) for sparking my interest in HDMI
	* https://www.fpga4fun.com/HDMI.html
* Bureau of Indian Standards for free equivalents of non-free IEC standards 60958-1, 60958-3, etc.
* @glenwing for [links to many VESA standard documents](https://glenwing.github.io/docs/)
