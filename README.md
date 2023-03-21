# rp2040-hdmi-1080p-hardware

Microcontrollers have begun to rival early PCs and retro gaming consoles in measures like speed and memory capacity, and even leapfrog them with new capabilities like multiple cores and wireless connectivity.  One area that still seems a bit lacking is video output.  Sure, the ability to connect a smallish LCD via i2c or SPI is normal, but can these little marvels do more?  With a little spicier circuit, I think so...

And, in fact, we know so: Wren6991 successfully bitbanged DVI on the RP2040 [ https://github.com/Wren6991/PicoDVI ] and validated the output up to 372Mbps (good enough for HD 720 @ 30Hz).  That leaves some resolution milestones unxplored, for example FHD 1920 x 1080 @ 60Hz (1080p) x 24 bit color depth (i.e. 8 bits for each of the red, green, and blue subpixels, aka RGB888) = 2.985984Gbits/second not including control signals and blanking overhead -- approaching an order of magnitude larger.  Back to Wren6991: the CPU cycles for QVGA (320 x 240) consume 60% of one of the RP2040's two M0+ cores, i.e. around 30% of its total processing power, so that means there's only headroom to about triple the processing rate.  Maybe with a very aggressive overclock it could get in the ballpark, but it seemed like, considering we have up to 30 GPIO pins to play with and DVI's Transition Minimized Differential Signalling (TMDS) encoding uses only 6 pins, there had to be a way of using the 32-bit width to help escape the bottleneck.  The RP2040's M0+ cores can execute 1 32-bit move instruction in 1 clock cycle, which means that if all 30 GPIO pins were tasked with sending this output, the theoretical maximum output from an ideal 1 core x 1 ipc code x 30 pins x 133Mhz 'normal' clock speed is around 3.99Gbits/second, although GPIOs are needed for other functions and some processing is necessary to work out what to send (and note that only 26 GPIO pins are exposed in the RPi Pico).  I considered an overclock and a Ghz-speed capable shift register, specifically the very interesting MC100EP142FA.  I considered a reduced color bit depth, like RGB565 or RGB666.  Ultimately, it sememed like an RGB to HDMI encoder would be the best idea to offload the encoding functions, and one can be had cheaper than the ultra-fast shift registers.

Another problem: with only 260KB of RAM onboard, how would the RP2040 store even a single frame that is 1920 x 1080 x 24 bits/pixel = 49,766,400 bits / 8 bits/byte = 6.220800 MB?  Display protocols require the frame transmitted many times per second.  Clearly the RP2040 needs another ic in the mix: external RAM >= 49.7664Mbits.  With this, the RP2040 could store the entire desired output for the current frame at all times, which frees the RP2040 from 'forgetting' and redrawing sections of each frame, i.e. it only has to deal with pixels that need to be updated and it can write those updates into memory over time, sending the current state of the frame in memory each time the video protocol needs a frame.  The best part about an independent memory chip is that the RP2040 doesn't necessarily need the data returned to it (supposing the use case doesn't require acting upon the existing frame, but simply overwriting it).  The RP2040 only needs to send the RAM the address of the next batch of pixels, which the RAM can then make available to the HDMI encoder/transmitter between their respective outputs and inputs.

When considering different RAM module options: static RAM (SRAM) would have the least timing hassles, but becuase they are often 16- or 8-bit (and exclusively so from JLCPCB at the time of working out the circuit; 24-bit options were available at Digikey), you would potentially need multiple RAM 'ranks' to alternate between to keep your reads continuous and have time to write.  Dynamic RAM (DRAM/SDRAM) has more timing overhead considering the refreshes DRAMs require, but there are some inexpensive options in the right size, speed, and bit widths for such a frame buffer.  A RAM ic with 'full page' burst read capability can spare you a lot of address transmissions and let you do some other things during those clock cycles, like prepare the next address or the next chunk of data to be written to the frame buffer.  I originally set the address info to be transmitted by a binary counter, but ultimately went with shift registers for more control from the software.

So, finding desktop resolution video output theoretically possible, I put together an RP2040 'desktop' board, complete with ethernet, USB-C in with 'real' (negotiated) power (not the USB A-to-C requirement or resistor hacks), a blingy gold sink board HDMI output with video and sound, i2s audio also piped to an onboard speaker, as well as a microphone input, two USB A for mouse and keyboard, a headphone jack, and hardware cutoff switches for mic and speaker, as well as compatibility and quality of life options.  Perhaps a good dev kit if you want to build on FreeRTOS, Zephyr, etc. -- by the way I love the maniac that made PyDOS for the RP2040 [ https://www.youtube.com/watch?v=Az_oiq8GE4Y ] and you should definitely watch his videos.

However, the board is pricy at $40 per unit via JLCPCB.  I scaled that back to just the 'proof of concept' stuff -- a cheaper USB-C setup (with the USB A-to-C requirement), no sink board connectors, no speaker or mic, no configuration switches.  This cut the build cost about in half.

If you had the hardware in hand, what would RAM addressing look like?  With multiple RAM ic's, supposing your frame buffer has pages of 512 bits [ see Micron's explanation of page size here https://www.micron.com/-/media/client/global/documents/products/data-sheet/dram/ddr4/4gb_ddr4_dram_2e0d.pdf -- basically page size = 2 * # of column address bits x # of DQ bits/8] that means with 49,766,400 bits per frame / (512 bits per page * 3 parallel 8-bit colors) =  32,400 pages to address / 2 ranks (i.e. you are swapping between two banks of either 2x16-bit or 3x8-bit SRAM; perhaps not ideal, becuase using CS to deselect ranks means a chip will not listen for writes while it is not being read -- it gets complicated when there are multiple ic's trying to talk on the same line) = 16,200 row addresses needed [if all in one bank, so you could just tie your bank select pins high or low], therefore address space bits needed: 2^14 = 16,384, so 14 bits are needed (advancing +1 every ~512 clock ticks).  Double-checking: 16,200 addresses * 512 bits * 3 parallel colors in high/low bits or independent 8-bit chips * 2 ranks = 49,766,400 bits (the math works out).  Check your datasheet for your bank, row, and column arrangement.

For control, you'll need at least RAS, CAS, and WE on the RAM.  You might be able to get away with tying CS and CKE to the values that keep the chip 'on,' especially if you're only using rank = 1, i.e. you are swapping between banks in the same chips, not between different sets of chips -- in this case, and if you're using DRAM you'll want to wire up your microcontroller to output to bank address pin(s) and whichever upper address bit, often A10, is used for setting auto-precharge.

For the shift registers used for writes to RAM (i.e. data), in addition to serial data in (and between registers, where you pipe the last output of one to the input of the next) you'll want to control their clocks, reset, and output enable (to set their outputs in a high impedance (high 'z') state, so they are not outputting their values on the same line the HDMI encoder is supposed to be reading from the RAM).  Or you can limit your writes to the HDMI blanking intervals, where the data input enable (DE) pin of the HDMI is set to ignore inputs.

As mentioned, SDRAM timing can be a pain.  For example, starting up might require a timed pause, triggering the data mask (DQM), precharging all banks, and running several (e.g. 8) auto refresh cycles.  During operation, you'll have to pay attention to some other timing details.  For example, if your max bank active time is 100,000 ns, you might hit a limit of 32 column reads per bank activation.  And then some time, in the low single digits as a percentage, is 'lost' to refreshing.  Check your datasheet for the refresh timing to calculate the overhead, especially if your use case timing requirements are close to the performance limits of your DRAM, e.g. some HDMI encoders operate at 165Mhz and many SDRAM banks operate at 166Mhz.


So what does everyone think?

Maybe something like this would be of interest to retro gamers wanting to plug their RP2040-based emulator into a TV/monitor.  Maybe piping a terminal to modern displays via hardware?  As mentioned, PyDOS is interesting.  Thinking about a GUI, the most feature complete one for microcontrollers I see out there is FabGL [ https://github.com/fdivitto/FabGL ] for the ESP32 with no clear option for the RP2040.  If you're after 1024 x 768 @ 60 FPS or smaller, 'going bigger' with an Allwinner V3S has RAM in the SIP and outputs RGB888.  For digital signage, a low end orange/banana/mango {insert tropical fruit here}/Rock pi/RADXA zero variant would be about the same price and validated.  Sometimes it's just fun to explore the limits.   I'm putting the board schematics and PCB files out there.  If you do use these, please double-check EVERYTHING, as I have not had these built/tested!
