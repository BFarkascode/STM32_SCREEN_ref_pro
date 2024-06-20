# STM32_SCREEN_ref_pro
Reference/base project for a bare metal screen driving project on a STM32_F429_DISCO board. This is the first project in a sequence of projects aimed.  

## Disclaimer
The code shared in this project is simply a slightly modified version of the solution from ControllersTech/Helentronica, allowing me to run it on my DISCO board. I wish not to take credit for it. 

## General description
How I tend to tackle truly complex projects is by starting with something very simple – call it a “base” project – then iterate from there. This helps with debugging and ensures that I can always be sure that the complexity one level removed from where I am, used to work without a hitch. No need to go back and try to debug something that is lower complexity to where I am working.
With that explained, let’s get to it, shall we?
If someone has spent some time in my repos already, he might have already come across the DE0Nano Digicam project I have done a few years ago (yes, documentation still pending on it, I know). That project used an FPGA to extract information (image) from a camera, then store that information on an SDcard. For what its worth, it was a great project and hugely complicated one with the fundamental aim of scalability and adaptability (i.e. to be capable to drive multiple cameras and acquire multiple megapixel-sized images). Nevertheless, I always wondered if it would be possible to do the same in a way that is “quick and simple”, relying on the resources available within a microcontroller only. More precisely, I was wondering if I could replicate the project cooked up in the dungeons at Adafruit (https://learn.adafruit.com/adafruit-ov7670-camera-library-samd51/overview) using a micro as small as possible, as little resources as possible and as STM32 as possible.
For a project like this, the “base” project would be a microcontroller publishing a simple bitmap to a screen using as little resources as possible. Following such a successful base project, I would change drivers, manipulate the screen input at my leisure as well as add speed to the setup so as to gain a refresh rate of, at least, 60 Hz. I would also want to add the OV7670 camera from the DE0Nano project and then synchronize everything to gain a 320x240 resolution, low latency screen feedback from the attached camera.
As the first part of this project, below I am presenting the base project using an STM32F429_DISCO board. Of note, I am new to the hardware and will be learning it on the fly.

## Toes dipped 
First thing one does with a brand-new hardware is to power it up and see what happens. Well, expectations would be that the device would work, and one could check, what the hardware is being capable of…but ST being ST, the example is of course bugged: the screen and the touch sensing is inverted meaning that touching the “top” of the screen is detected as touching the “bottom”. Annoying.
Okay, no problem, let’s go to the provider’s examples library, fetch a project that one fancies…then slam dunk the example on the hardware and see what happens. Yeap, same bug.
Okay then, let’s do the long way around. Go through a basic the tutorial for the TouchGFX engine (ST’s gui generating solution), generate a simple turn on LED solution (for instance, this one from CotnrollersTech: https://www.youtube.com/watch?v=rGxDZPYcWIs&list=PLfIJKC1ud8giOsk-C4BCOwSHtbXqTNb1W&index=1&pp=iAQB) and then enjoy.
Bug is still there. Generated using official code from the official sites using the official tools on an official hardware. Times like these always remind me why I started to avoid using officially provided HAL code for any of my designs and go bare metal where possible…
(Just an FYI, the bug is within the “STM32TouchCotnroller.cpp” file’s “bool STM32TouchController::sampleTouch” function where the touch position is aligned with the screen. Replace the Y axis line with “y = 300 - state.Y;” to get rid of it. My guess the example was written for a board revision that is not the same as the one available these days on the market.) 
At any rate, for investigating the hardware and the software…and to figure out, what the hell is going on, the bug is irrelevant. We won’t be using the touch screen anyway. What matters is that we have a screen working with a gui that we have designed in the TouchGFX software. We also have a code and an “.ioc” configuration file generated for us by TouchGFX.
Now, we can open CubeMX – or the CubeMX plugin of the IDE – and inspect the list of resources we are using for this particular project by exploring the “Pinout & Configuration” of the project. We have GPIO, NVIC, RCC, SYS, TIM6, FMC, I2C3, SPI5, DMA2D, LTDC, CRC, FREERTOS, X-CUBE-TOUCHGFX in green. That’s a lot of stuff…

### What are all these???
I don’t want to touch upon GPIO, NVIC, RCC, SYS, TIM6, I2C3, SPI5 and CRC, since they should be rather clear from previous projects.
On the other hand, we have a few newcomers that may not have been seen before:
- FMC is the “flexible memory controller”. As far as I can tell, FMC is a type of hardware DMA specifically made to facilitate parallel serial interfaces that are crucial for driving SDRAMs or LCD screens. As such, it physically connects the external hardware to the MCU. The FMC could drive either the LCD screen or the SDRAM though in the project TouchGFX has generated for us, it is doing the interfacing towards the external 8 MB SDRAM. The SDRAM is crucial due to memory restrictions (see below).
- DMA2D is the “ChromArt” two dimensional DMA specifically made to help pushing images from one place to the other. Like all DMAs, it allows data transfer without any MCU overhead, making the hardware significantly faster.
- LTDC is the 16 (or 18) bit parallel LCD screen writing hardware. Similar to the FMC, it’s job is to parallel-publish data.
- FREERTOS is the go-to real time operating system for MCUs. It allows parallel computing, meaning that our main code could run the same time as the graphic interface is updated. This is of course pretty useful and decreases latency between user input and the device doing stuff. FREERTOS is a “middleware”, a code section that is external to our main loop.
- X-CUBE-TOUCHGFX is the graphic engine generated and implemented into our code by the TouchGFX software. It is a “middleware” and is external to the main loop. It also has some read-only elements which we will not be able to tinker with later using the IDE.
### A problem of excess?
One might wonder, why it is necessary to have so much stuff running at the same time to just publish a simple image to a screen. Well, it isn’t. Simply put, the project we have generated is optimized for speed and utility already, which means that it is far from the kind of “simple”, “low resource” project we are looking for as our base.
After reading a bit more and exploring the configuration, it became clear to me that:
- DMA2D could be removed from the project and things would still work fine. DMA2D is just there to make our frame buffer be published without MCU overhead
- FREERTOS is there to allow the TouchGFX engine run parallel to our main code. If we have only one image – no transitions, no updates demanded from the TouchGFX engine – we should not need an RTOS.
- LTDC is running, but there is a much simpler – and slower – method to drive the LCD by using SPI.
- FMC – and the connected SDRAM – are not necessary for simple image publishing projects. There is a workaround when using SPI to communicate with the LCD (see below).
- I2C3 runs the gyroscope and the touch sensor and the gyroscope. We don’t need them for our base.
With all that, I decided to trim the fat from the project generated by TouchGFX…only to immediately hit a brick wall and be reminded, yet again, why I dislike officially generated code. As it goes, the project we currently have assumes that all functions within our DISCO board will be used and populates the configuration classes as such. Manually removing anything would mean tracking down all instances of all unnecessary elements and removing them while also ensuring that the configuration matrices are properly populated with empty values. This is just way too much of a tedious work, especially considering how the IDE doesn’t seem to be able to properly track code elements within the middleware sections…not to mention how the RTOS makes debugging this project within its current state practically impossible.
No, in order to proceed forward, we would need to build the project above from scratch…

### Memory allocation
Before we go back to the drawing board, I want to talk a bit about memory allocation within the project above. We are storing (or, well the TouchGFX engine is storing) our images/screens in 2D arrays or 16-bit RGB565 colour pixels called bitmaps. We need 320x240x2 = 153600 bytes to store the entire bitmap of a full screen (assuming 16 bit colour depth RGB565). That’s our so called “frame buffer”. Of note, the TouchGFX is screen agnostic and doesn’t care what happens to this frame buffer after it has been updated by the engine.
In the project, this is then tripled to allow smooth screen transitions - original screen, new screen, animation between the two. That’s 450 kB of image data, waaaay over the RAM limit we have in our F429ZI (we have 200 kB of RAM only). Unfortunately, even one frame buffer would exceed the limit of RAM due to the additional code overhead coming from, well, the actual code.
The SDRAM is thus there to hold the three frame buffers. (I am not sure, why FLASH could not be used for the same purpose, maybe it is a question of speed or that the boffins at ST weren’t keen on using FLASH for such purposes…)
Luckily, there is a workaround to the memory problem when using SPI to drive the screen (and ONLY with SPI!): we can publish the frame buffer to the screen in conseguitive blocks! These blocks have a maximum size of 20 lines (altogether 12800 bytes) and thus allow us to let go of the SDRAM and the FMC. (Again, I am not sure why this capability isn’t available with LTDC and FMC screen driving, nor do I know why the block size is limited to 12800 bytes. Probably again a configuration thing at ST where they just didn’t think about it – or cared).

###Screen drive anomaly
There is one last thing that I wish to mention and that is that I still struggle to figure out, what is actually driving the LCD screen in the project. Investigating the code indicates that we are using SPI (reinforced by the fact that the screen driver ILI9341 ICs drive selector IM[0:3] pins are hard-wired to be 4’b0110 which corresponds to SPI driving) yet hooking up an oscilloscope to the LTCD output pins, we have traffic there (at 5.5 MHz). The ILI9341 datasheet suggests that these pins should all be pulled to GND when we are in SPI mode, which clearly isn’t the case. (In the eventual base project I have made, these pins remain GND so it’s not the ILI9341 that is “publishing backwards” either).
My guess is yet again that the example code was originally written for something else which does use LTDC to drive the screen, but the TouchGFX’s code generation was not updated properly. Instead somebody just manually rerouted the frame buffer to SPI and left all the other things activated. I have no proof of this, just a hunch from all the other bugs I have come across using official ST solutions.


## To read
I highly recommend checking the following youtube videos and read the related documents before attempting to create something from scratch:
-How to integrate TouchGFX in a custom board (The long way round) - YouTube
-How to set up TouchGFX with SPI Displays || ILI9341 - YouTube
-TouchGFX on a custom made low cost board with the ILI9341 controller over SPI – HELENTRONICA
It is also a good idea to familiarise oneself with the ICs and hardware elements available on our DISCO board. I don’t share the datasheets and the refmans here, but we have:
-STM32F429ZI_DISCO schematic
-STM32F249ZI MCU (datasheet and refman both needed)
-FRIDA_LCD_FRD240C48003-B screen
-ILI9341 screen driver

## Particularities

### The hardware unravelled
First are foremost, we need to figure out, how are hardware elements are wired on our DISCO board. Thankfully, we have access to the schematics of our DISCO board on ST’s site, so we can take note of the parts:
- STM32F249ZI is the MCU. This is a relatively low-end microcontroller, pretty much the smallest that can both drive an LCD using parallel and also interface parallel with a camera.
- FRIDA_LCD_FRD240C48003-B is the screen with ILI9341 as the integrated screen driver. Here we must pay attention to the different pins that change label between the screen and the MCU, such as “CSX” which turns into CS or the WR running into “WRX_DCX”. These pins are important for the SPI driving and we will need to implement them manually (see “ili9341.c”). Also, we can see from the schematic that the screen is connected to SPI5 so all drivers we will be implementing will have to point to this bus (see “ili9341.c” for the driver and “TouchGFX_DataTransfer.c” for the DMA callback function(!)). Also-also, as mentioned above, the ILI9341 controls pins (IM[3:0]) are hard-wired for 8-bit SPI control. To change that we would need to modify our board…Also-also-also, the LTDC pins are all connected to the screen, but we won’t be needing them.
- STMPE811QTR touch sensor on I2C3. We can ignore that since we won’t be using it. Same time, we can ditch setting up I2C3 in our new code.
- I3G4250D gyro is on SPI5. We can simply ignore it.
- IS42S16400J SDRAM on the FMC pins. We don’t need to implement FMC.

###Project generation bugs
As mentioned above, the code shared is a direct copy of what ControllersTech is doing in his youtube video.
He mentioned though that he could not make the project work in the IDE. Well, I did manage to do it, albeit by using some rather crude methods. As it goes, no matter how much we wish to implement the X-CUDE_TOUCHGFX middleware into a custom project, the project will refuse to generate it. We will be able to interact and update the part of the code using the TouchGFX software, so it is not the interfacing that is bugged. It seems to me that the CubeMX plugin is the culprit.
A workaround is to manually copy-paste the “Middleware” folder from another project to this one (the discarded example project is coming handy here) and then refresh the modified project tree. Don’t forget to add the path to this folder within Project->Properties->C/C++ General->Paths and Symbols so as to include it into the project! This workaround is very crude and will need to be redone every time we generate a new code using the CubeMX.

## User guide
One can simply follow the youtube video from ControllersTech to make use of this code.

## Conclusion
Now that we have a nice working base project, we can start modifying (breaking) things.

