# STM32_SCREEN_ref_pro
Reference/base project for a bare metal screen driving project on a STM32_F429_DISCO board. This is the first project in a sequence of projects planned.  

## Disclaimer
The code shared in this project is simply a slightly modified version of the solution from ControllersTech/Helentronica, allowing me to run it on my DISCO board. I wish not to take credit for it. 

## General description
How I tend to tackle truly complex projects is by starting with something very simple – call it a “base” project – then iterate from there. This helps with debugging and ensures that I can always be sure that the complexity one level removed from where I am, used to work without a hitch. No need to go back and try to debug something that is lower complexity to where I am working.

With that explained, what is on the table today?

If someone has spent some time in my repos already, he might have already come across the DE0Nano Digicam project I have done a few years ago (yes, documentation still pending on it, I know). That project used an Cyclone IV FPGA to extract image information  from a camera, then store that information on an SDcard. For what its worth, it was a great project but a grosly overcomplicated one with the fundamental base requirement of scalability and adaptability (i.e. to be capable to drive multiple cameras and potentially acquire multiple megapixel-sized images). It also got quickly out of hand...

Anyways, I always wondered if it would be possible to do the same in a way that is “quick and simple”, relying on the resources available within a microcontroller only and not run an FPGA. After all, MCUs also have in-built parallel capture and pbulish capabilities. More precisely, I was wondering if I could replicate the project cooked up in the dungeons at Adafruit (their SAMD51 camera drive project) using a micro as small as possible, as little resources as possible...using an STM32.

What does that leaves us? Well, for a project like this, the “base” project would be a microcontroller publishing a simple bitmap to a screen using as little resources as possible (no fancy engines or advanced DMAs). Once done, I could iterate from there by changing drivers, manipulate the screen input at my leisure as well as add speed to the setup so as to gain a refresh rate of, at least, 60 Hz. I would also be able to add the OV7670 camera from the DE0Nano project and then synchronize everything to gain a 320x240 resolution, low latency screen feedback from the attached camera.

Let's get to it then!

## Toes dipped 
I picked the STM32F429_DISCO board for the project since this was the cheapest available solution (about 30 EUROs as of time of writing) on the market that has both LTDC parallel LCD screen capability and the DCMI parallel camera capture. This is a new peace of hardware to me, so bear with me, I will be learning it on the fly...

First thing one does with a brand-new hardware is to power it up and see what happens. Expectations would be that the device would work, and one could check, what the hardware is being capable of...

ST being ST, the example is of course bugged. The screen and the touch sensing is inverted meaning that touching the “top” of the screen is detected as touching the “bottom”.

Okay, no problem, let’s go to the ST’s examples library, fetch a project that is similar, then slam-dunk it on the hardware and see what happens.

Yeap, same bug.

Okay then, let’s do the long way around. Go through a basic the tutorial for the TouchGFX engine (which is ST’s gui generating solution, more on it later), generate a simple turn on LED solution (for instance, the first video on TouchGFX from ControllersTech) and then enjoy...

...the bug being still there.

To recap, generated using official code from the official sites using the official tools on an official hardware, the result is bugged. Times like these always remind me why I started to avoid using officially provided HAL code for any of my designs and go bare metal where possible…

(Just an FYI, the bug is within the “STM32TouchController.cpp” file’s “bool STM32TouchController::sampleTouch” function where the touch position is aligned with the screen. Replace the Y axis line with “y = 300 - state.Y;” to get rid of it. My guess the example was written for a board revision that is not the same as the one available these days on the market.) 

No matter. For investigating the hardware and the software and to figure out, what the hell is going on under the hood of the DISCO board, this bug is irrelevant. We won’t be using the touch screen anyway. What matters is that we have a screen working with a gui that we have designed using the TouchGFX software.

We also have a code and an “.ioc” configuration file generated for us by TouchGFX. We can open the ".ioc" file using CubeMX – or the CubeMX plugin of the IDE – and inspect the list of resources we are using by exploring the “Pinout & Configuration” part of the file.

In short, we have GPIO, NVIC, RCC, SYS, TIM6, FMC, I2C3, SPI5, DMA2D, LTDC, CRC, FREERTOS, X-CUBE-TOUCHGFX in green and thus, being used. That’s a lot of stuff.

### What are all these?
I don’t want to go into GPIO, NVIC, RCC, SYS, TIM6, I2C3, SPI5 and CRC, since they all be familiar from previous STM32 projects.

On the other hand, we have a few newcomers:
- FMC is the “flexible memory controller”. As far as I can tell, FMC is a type of hardware DMA specifically made to facilitate parallel serial interfaces that are crucial for driving SDRAMs or LCD screens. It pretty much physically connects the external hardware to the MCU. The FMC could drive either the LCD screen or the SDRAM though in the project we have generated using TouchGFX, it is doing the interfacing towards the external 8 MB SDRAM. OF note, the SDRAM is crucial due to memory restrictions (see below).
- DMA2D is the “ChromArt” two dimensional DMA. It was specifically made to help pushing images from one place to the other. Like all DMAs, it allows data transfer without any MCU overhead, making execution significantly faster.
- LTDC is the 16 (or 18) bit parallel LCD screen interface hardware element. Similar to the FMC, it’s job is to parallel-publish data.
- FREERTOS is the go-to real time operating system for MCUs. It allows parallel computing, meaning that - in our immediate example - the main code could run the same time as the graphic interface. This is of course pretty useful and decreases latency between user input and the device. FREERTOS is a “middleware”, a code section that is external to our main loop.
- X-CUBE-TOUCHGFX is the graphic engine generated and implemented into our code by the TouchGFX software. It is a “middleware” and is external to the main loop. It also has some read-only elements which we will not be able to tinker with later using the IDE.

### Is there a problem of excess?
One might wonder, why it is necessary to have so much stuff running at the same time to just publish a simple image to a screen. Aren't we just overcooking this? Well, yes, we do. Simply put, the project we have generated is optimized for speed and utility already, which means that it is far from the kind of “simple”, “low resource” project we are looking for as our base project.

For example, after reading a bit more and exploring the configuration, it became clear to me that:
- DMA2D could be removed from the project and things would still work fine. DMA2D is just there to make our frame buffer be published without MCU overhead, but if we don't care about MCU load, we don't need DMA2D.
- FREERTOS is there to allow the TouchGFX engine run parallel to our main code. If we have only one image – no transitions, no updates demanded from the TouchGFX engine – we do not need an RTOS.
- LTDC is running, but there is a much simpler – and slower – method to drive the LCD by using SPI. We should be looking into that first.
- FMC – and the connected SDRAM – are not necessary for simple image publishing projects. (There is a workaround when using SPI to communicate with the LCD, see below).
- I2C3 runs the touch sensor and the gyroscope. We don’t need them now.

With all that, I took the reasonable decision to "trim the fat" from the project generated by TouchGFX by removing what I don't need...only to immediately hit a brick wall and be reminded, yet again, why I dislike officially generated code. As it goes, the project we currently have assumes that all functions within our DISCO board will be used and populates the configuration classes as such. Manually removing anything would mean tracking down all instances of all unused elements and removing them while also ensuring that the configuration matrices are properly repopulated . This is just way too much of a tedious work, especially considering how the STM32 IDE doesn’t seem to be able to properly track code elements within the middleware sections.

No, in order to proceed forward, we would need to build our base project from scratch.

### Memory allocation
Before we go do that, I want to talk a bit about memory allocation within the project above because it is crucial for understanding, what is going on. We store (or, well the TouchGFX engine is storing) our images/screens in 2D arrays or 16-bit RGB565 colour pixels called bitmaps. We need 320x240x2 = 153600 bytes to store the entire bitmap of a full screen (assuming 16 bit colour depth RGB565). This is our so called “frame buffer”.

In the project, this is then tripled to allow smooth screen transitions - original screen, new screen, animation between the two. That’s 450 kB of image data, waaaay over the RAM limit we have in our F429ZI (we have 200 kB of RAM only). Unfortunately, even one frame buffer would exceed the limit of RAM due to the additional code overhead coming from, well, the actual code. The SDRAM is thus there to hold the three frame buffers and is seemingly crucial. (I am not sure, why FLASH could not be used for the same purpose, maybe it is a question of speed or that the boffins at ST weren’t keen on using FLASH for such purposes…)

Of note, TouchGFX engine is screen agnostic and doesn’t care what happens to the frame buffers after the have been updated.

Luckily, there is a workaround to the memory problem when using SPI to drive the screen (and ONLY with SPI!): we can publish the frame buffer to the screen in conseguitive blocks! These blocks have a maximum size of 20 lines (altogether 12800 bytes) and thus allow us to let go of the SDRAM and the FMC. (Again, I am not sure why this capability isn’t available with LTDC and FMC screen driving, nor do I know why the block size is limited to 12800 bytes. Probably again a configuration thing at ST where they just didn’t think about it – or cared). We will be using this in our base project.

### Screen drive anomaly
There is one last thing that I wish to mention. Frankly, I still struggle to figure out, what is actually driving the LCD screen in the example project we have originally done. Investigating the code indicates that we are using SPI (reinforced by the fact that the screen driver ILI9341 ICs drive selector IM[0:3] pins are hard-wired to be 4’b0110 which corresponds to SPI driving) yet hooking up an oscilloscope to the LTCD output pins, we have traffic there (at 5.5 MHz). The ILI9341 datasheet suggests that these pins should all be pulled to GND when we are in SPI mode, which clearly isn’t the case. (n the eventual base project I have made following ControllersTech, these pins remain GND so it’s not the ILI9341 that is “publishing backwards” either.

My guess is yet again that the example code was originally written for something else which does use LTDC to drive the screen, but the TouchGFX’s code generation was not updated properly and that code section sort of got "stuck" in the example. The frame buffer got manually rerouted to SPI. I have no proof of this, just a hunch from all the other bugs I have come across using official ST solutions.


## To read
I highly recommend checking the following youtube videos and read the related documents before attempting to create something from scratch:
- How to integrate TouchGFX in a custom board (The long way round) - YouTube
- How to set up TouchGFX with SPI Displays || ILI9341 - YouTube
- TouchGFX on a custom made low cost board with the ILI9341 controller over SPI – HELENTRONICA

It is also a good idea to familiarise oneself with the ICs and hardware elements we will be using on the DISCO board. I don’t share the datasheets and the refmans here, but we have the following ICs:
- STM32F249ZI MCU (datasheet and refman both needed)
- FRIDA_LCD_FRD240C48003-B screen
- ILI9341 screen driver

## Particularities

### The hardware unravelled
First are foremost, we need to figure out, how the hardware elements are wired on our DISCO board. Thankfully, we have access to the schematics of our DISCO board on ST’s site, so we can take note of the parts:
- STM32F249ZI is the MCU. This is a relatively low-end microcontroller, pretty much the smallest that can both drive an LCD using parallel and also interface parallel with a camera.
- FRIDA_LCD_FRD240C48003-B is the screen with ILI9341 as the integrated screen driver. Checking the schematic, we must pay attention to the different pins that change label between the screen and the MCU, such as “CSX” which turns into CS or the WR running into “WRX_DCX”. These pins are important for the SPI driving and we will need to implement them manually (see “ili9341.c”). Also, we can see from the schematic that the screen is connected to SPI5 so all screen drivers will have to point to this bus (see “ili9341.c” for the driver and “TouchGFX_DataTransfer.c” for the DMA callback function(!)). Also-also, as mentioned above, the ILI9341 controls pins (IM[3:0]) are hard-wired for 8-bit SPI control. To change that we would need to modify our board.
- STMPE811QTR touch sensor on I2C3. We can ignore that since we won’t be using it. Same time, we can ditch setting up I2C3 in our new code completely.
- I3G4250D gyro is on SPI5. We can simply ignore it.
- IS42S16400J SDRAM on the FMC pins. We don’t need to implement FMC. Ignore.

### Project generation bugs
As mentioned above, the code shared is a direct copy of what ControllersTech is doing in his youtube video.

He mentioned though that he could not make the project work in the IDE. Well, I did manage to do it, albeit by using some rather crude methods. As it goes, no matter how much we wish to implement the X-CUDE_TOUCHGFX middleware into a custom project, the project will refuse to generate it. We will be able to interact and update the part of the code using the TouchGFX software, so it is not the interfacing that is bugged. It seems to me that the CubeMX plugin is the culprit.

A workaround is to manually copy-paste the “Middleware” folder from another project our news one (the discarded example project is coming handy here) and then refresh the modified project tree. Don’t forget to add the path to this folder within Project->Properties->C/C++ General->Paths and Symbols so as to include it into the project! This workaround is very crude and will need to be redone every time we generate a new code using the CubeMX.

## User guide
One can simply follow the youtube video from ControllersTech to make use of this code.

## Conclusion
Now that we have a nice working base project, we can start modifying (breaking) things!

