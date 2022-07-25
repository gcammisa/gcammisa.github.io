---
layout: post
title: Bad Apple, but i wanted it to play on a Raspberry Pi Pico using micropython to control a led matrix
---

A couple months ago I bought some of the (at the time) new Raspberry Pi Pico from aliexpress.
Coincidentally, I also bought some 2 16x16 and 1 8x32 led matrices.
And I obviously had no real use for all these things that I impulse-bought.

So I needed some kind of project to:  
1. Justify buying stuff that I didn't need
2. Try out micropython. The idea of using python on a microcontroller seems kind of *_interesting_* to me.

The question now is: what kind of project?  
And, me being me, there is only one answer: **BAD APPLE!**
There will never be enough devices running bad apple in this world.

Bad apple is a really nice video to play on led matrices because it only needs 2 colours and is recognizable even when played at low resolution.

Let's start by laying out some requirements:  
1) The video should be playing at full speed
2) No input from an external computer / device should be needed, everything has to be self-contained into the Pi Pico

This means that we need:  
1. A 32x24 version of bad apple small enough to fit into the Pi Pico onboard memory
2. Code fast enough to read a frame and push it to the led matrices at least 15 times each second. Python on a microcontroller sounds slow to me, so this might not be as easy as it sounds.

Let's start by downloading a bad apple video from youtube.
I used youtube-dl, but you can use whatever you want.
Now we need to extract the frames from this video.
Again, you can use whatever tool you want, but I personally used old reliable ffmpeg.
	
Now we have a bunch of pictures (and we don't need half of them, we're aiming for 15fps, not 30fps).
But we need to do some more preparation before we can use them:
1. The resolution is obviously waaaay too big for our 32x24 led matrix
2. These pictures are not really using only 2 colours: there are shades of grey at the edges

So I set up a batch job in photoshop that does this:
1. Convert the image to indexed color, with a 2 color palette (black and white)
2. Resize the image to 32x24
3. Save it as a BMP

Now we're ready to "convert" the frames in a format of our liking that we can then read on the Pi Pico.
If we want our file to be able to be played by the Pi Pico with the minimal amount of processing possible, in this phase of our work we need to already account for how the matrices are wired.
In my setup the matrices are wired in series, and each matrix is wired as if it was a long led strip, with the even columns going from top to bottom and the odd columns going from bottom to top. 

After some trial and error I chose to convert each pixel to 1 bit (1 = black, 0 = white), group them in chunks of 4 and encode each chunk as an HEX char in a text file that we can then load into the Pi Pico memory.
There are many better ways to do this (like just using a simple binary file, or something like RLE compression), but I didn't deem it necessary for this "POC".

So what we need is a python script that reads all our bitmaps containing the frames, writes a 1 or 0 for each pixel of each bitmap, rearranges the bits to account for the wiring of the matrices and then saves the output as hex text.
I did this in two different "stages": one script to convert the bitmaps to binary text and another script to convert that to hex text
The scripts are extremely ugly, but they work and we only need to run them once.
Code to convert the bitmaps to binary text:
```python
from PIL import Image
import numpy as np

binarystring = ''

for filenum in range(6570):
	filename = 'img' + str(filenum+1).zfill(4) + '.bmp'
	im = Image.open(filename)
	p = np.array(im)


	for i in range (0,32):
		tempcolumn = []
		for j in range (0,16):
			tempcolumn.append(str(p[j][i]))
		if i%2 != 0:
			for k in range(len(tempcolumn)-1, -1, -1):     
				intlist = list(map(int,tempcolumn[k].replace('[','').replace(']','').split()))
				if (sum(intlist) < 383):
					binarystring += '0'
				else:
					binarystring += '1'
		else:
			for k in range(0, len(tempcolumn)):    
				intlist = list(map(int,tempcolumn[k].replace('[','').replace(']','').split()))
				if (sum(intlist) < 383):
					binarystring += '0'
				else:
					binarystring += '1'


splitbinarystring = [binarystring[i:i+8] for i in range(0, len(binarystring), 8)]

data = b''
for elem in splitbinarystring:
	data += bytes([int(elem,2)])

with open('badapple.bin', 'wb') as f:
    f.write(data)
```

Code to convert the binary text to hex text:
```python
import array, time
bits = { '0':'0000', '1':'0001', '2':'0010', '3':'0011',
         '4':'0100', '5':'0101', '6':'0110', '7':'0111',
         '8':'1000', '9':'1001', 'A':'1010', 'B':'1011',
         'C':'1100', 'D':'1101', 'E':'1110', 'F':'1111' }

def hex2bin(hexstring):
    r = ""
    for c in hexstring:
        r += bits[c]
    return r

fw = open('output.txt', 'a')
f = open('hexapple.py', 'r')
count = 0

while True:
    count += 1
 
    # Get next line from file
    frame = f.readline()
 
    # if line is empty
    # end of file is reached
    if not frame:
        break
        
    fw.write(hex2bin(frame[:-1]))
    fw.write("\n")
f.close()
```

It's ugly and kinda slow, but it gets the job done and we need to run it only once.

Now we need to write some micropython code to control our led matrices, which is basically a long folded string of WS2812b LEDs.
Somebody for sure has already used WS2812b with the pi pico, right? Right.
I don't think reinventing the wheel is a smart thing to do (usually), so let's start from [THIS](https://core-electronics.com.au/tutorials/how-to-use-ws2812b-rgb-leds-with-raspberry-pi-pico.html) example that I found online, add the features that we need (reading from file, parsing the frame) and remove the ones that we don't need
To handle I/O the Pi Pico has feature called PIO, which is basically a programmable coprocessor dedicated to handling I/O. 
The PIO ASM code to handle the WS2812b protocol is already implemented in the example we're starting from, so we don't have to do it, but PIO seems like an interesting feature to me and I'll probably experiment more with it in the future.
https://blues.io/blog/raspberry-pi-pico-pio/

Here is the commented code that I ended up using for the "final" version of this:

```python
# Example using PIO to drive a set of WS2812 LEDs.
import array, time
from machine import Pin
import rp2

# Configure the number of WS2812 LEDs.
NUM_LEDS = 768
PIN_NUM = 22
brightness = 0.1

@rp2.asm_pio(sideset_init=rp2.PIO.OUT_LOW, out_shiftdir=rp2.PIO.SHIFT_LEFT, autopull=True, pull_thresh=24)
def ws2812():
    T1 = 2
    T2 = 5
    T3 = 3
    wrap_target()
    label("bitloop")
    out(x, 1)               .side(0)    [T3 - 1]
    jmp(not_x, "do_zero")   .side(1)    [T1 - 1]
    jmp("bitloop")          .side(1)    [T2 - 1]
    label("do_zero")
    nop()                   .side(0)    [T2 - 1]
    wrap()


# Create the StateMachine with the ws2812 program, outputting on pin
sm = rp2.StateMachine(0, ws2812, freq=8_000_000, sideset_base=Pin(PIN_NUM))

# Start the StateMachine, it will wait for data on its FIFO.
sm.active(1)

# Display a pattern on the LEDs via an array of LED RGB values.
ar = array.array("I", [0 for _ in range(NUM_LEDS)])

##########################################################################
def pixels_show():
    dimmer_ar = array.array("I", [0 for _ in range(NUM_LEDS)])
    for i,c in enumerate(ar):
        r = int(((c >> 8) & 0xFF) * brightness)
        g = int(((c >> 16) & 0xFF) * brightness)
        b = int((c & 0xFF) * brightness)
        dimmer_ar[i] = (g<<16) + (r<<8) + b
    sm.put(dimmer_ar, 8)
    time.sleep_ms(10)

def pixels_set(i, color):
    ar[i] = (color[1]<<16) + (color[0]<<8) + color[2]

def show_frame_from_colorlist(colorlist):
    i = 0
    for color in colorlist:
        pixels_set(i, color)
        i = i+1
    pixels_show()

bitsToColor = { 
		'0':[(0,0,0),(0,0,0),(0,0,0),(0,0,0)],
		'1':[(0,0,0),(0,0,0),(0,0,0),(255,255,255,255)], 
		'2':[(0,0,0),(0,0,0),(255,255,255),(0,0,0)], 
		'3':[(0,0,0),(0,0,0),(255,255,255),(255,255,255)],
        '4':[(0,0,0),(255,255,255),(0,0,0),(0,0,0)], 
		'5':[(255,255,255),(255,255,255),(0,0,0),(255,255,255)], 
		'6':[(0,0,0),(255,255,255),(255,255,255),(0,0,0)], 
		'7':[(0,0,0),(255,255,255),(255,255,255),(255,255,255)],
        '8':[(255,255,255),(0,0,0),(0,0,0),(0,0,0)], 
		'9':[(255,255,255),(0,0,0),(0,0,0),(255,255,255)], 
		'A':[(255,255,255),(0,0,0),(255,255,255),(0,0,0)], 
		'B':[(255,255,255),(0,0,0),(255,255,255),(255,255,255)],
        'C':[(255,255,255),(255,255,255),(0,0,0),(0,0,0)], 
		'D':[(255,255,255),(255,255,255),(0,0,0),(255,255,255)], 
		'E':[(255,255,255),(255,255,255),(255,255,255),(0,0,0)], 
		'F':[(255,255,255),(255,255,255),(255,255,255),(255,255,255)] 
		}
def hex2color(hexstring):
    r = []
    for c in hexstring:
        try:
            r.extend(bitsToColor[c])
        except:
            pass
    return r

def badapple():
    f = open('badapple15fps.py', 'r')
    count = 0
    while True:
        count += 1
        # Get next line from file
        frame = f.readline()            
        show_frame_from_colorlist(hex2color(frame[:-1]))
    f.close()
    
if __name__ == "__main__":
    # execute only if run as a script
    badapple()
```

And here's the result! (It's a youtube video, click on the thumbnail!)

[![YOUTUBE VIDEO](http://img.youtube.com/vi/nImuWfNGEpQ/0.jpg)](http://www.youtube.com/watch?v=nImuWfNGEpQ "Bad Apple!")
