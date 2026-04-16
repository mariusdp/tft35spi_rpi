# Getting TFT35-SPI working on a raspberry pi

This works with different setups with the [TFT35-SPI](https://github.com/bigtreetech/TFT35-SPI/tree/master):
- rpi: wires directly connecting the JST pins on the display with the gpio pins of pi
- rpi: with [IO2CAN](https://github.com/bigtreetech/IO2CAN/tree/master) from bigtreetech. There, please plug the IO2CAN into the gpio headers of the raspberry pi, connect the ribbon cable to the display and IO2CAN and **set the switch on the IO2CAN to "CM4"**.
- CM4: with a FCC cable to the "SPI screen" connector on a Manta board with CM4

**Note**: You can also use the "cb1_armbian" branch to build the latest kernel driver from BTT for newer armbian versions. This removes the problem with the spurious/fake touch inputs on my Manta5/CB1 setup.
  
Pin out:
| TFT35-SPI | IOCAN gpio pin | Manta FCC gpio pin |
| -- | --- | --- |
| +5 V | any 5 V | comes from board |
| GND | any GND | comes from board |
| MISO_LCD (SPI MISO) | 9 | 9 |
| MOSI_LCD (SPI MOSI) | 10 | 10 |
| SCK_LCD (SPI SCLK) | 11 | 11 |
| NSS_LCD (SPI CS) | 7 | 4 | 
| RS_LCD | 17 | 17 |
| SDA_NS2009 (I2C SDA) | 27  | 27 |
| SCL_NS2009 (I2C SCL) | 22 | 22 |

The gpio pins are set up to use the *spi0* interface. In case of the CM4 with the Manta board, a different chip select (CS) pin is used while with IO2CAN the CS pin is mapped to the default *cs1 pin* of *spi0*. 


> [!NOTE]
> - With IO2CAN, the display uses the spi0-1 interface. However, the display seems to interfere with the CAN interface and I got corrupted packets and timeouts in klipper when homing. Lowering the bandwidth for the dispay did not help, so using CAN and the display does not seem to work reliably. Moving the display to other gpio pins is also not easy as the spi mode it uses is not supported on the other spi interfaces of rpi zero 2 or rpi 3.
> - You will have to re-build the kernel module for the tfttouch part every time your kernel changes.


## Setting up the display:

- add to the end of /boot/config.txt or on bookworm and higher /boot/firmware/config.txt:

with IO2CAN:
```
dtparam=spi=on
dtoverlay=fbtft,spi0-1,ili9486,width=320,height=480,dc_pin=17
dtparam=cpha=1,cpol=1,speed=24000000,fps=60
dtparam=bgr=1,rotate=90,txbuflen=307200
```
to setup the SPI interface and load the display driver.

_Please note that this uses spi0-1 and dc_pin=17 unlike written on the TFT35-SPI git._

with Manta/CM4:
 ```
dtoverlay=spi0-2cs,cs0_pin=23,cs1_pin=4,
dtoverlay=fbtft,spi0-1,ili9486,width=320,height=480,dc_pin=17
dtparam=cpha=1,cpol=1,speed=24000000,fps=60
dtparam=bgr=1,rotate=90,txbuflen=307200
``` 
On the CB1 spi0-0 is reserved for CAN with the Manta, so it might be useful to set it up as well.

- to get console output, append to the line in /boot/cmdline.txt or on bookworm and higher /boot/firmware/cmdline.txt after "rootwait":
```
 fbcon=map:10 fbcon=font:8x8 logo.nologo
```

- reboot and check if you get output. You can use ```dmesg``` to see if the module was loaded correctly.

## Setting up touch:

- update system and install the build tools:
```
sudo apt-get update
sudo apt-get dist-upgrade
sudo apt install git bc bison flex libssl-dev
```

- install the kernel headers based on your system
with rpi OS bookworm:
for 64 bit:
```
sudo apt install linux-headers-rpi-v8
``` 
or for 32 bit:
```
sudo apt install linux-headers-rpi-{v6,v7,v7l}
```

with rpi OS bullseye:
```
sudo apt install --reinstall raspberrypi-kernel-headers
```

- install this repo:
```
git clone https://github.com/stadie/btt_tft35spi_rpi.git btt_tft35
cd btt_tft35
```

- build the module using two source files for the tsc2007 module that were modified for the [CB1 kernel](https://github.com/bigtreetech/CB1-Kernel) and build an overlay that will load the driver:
```
cd tsc2007
make
```

- copy module to kernel modules and the new overlay to boot:
```
make install
```

- add to the end of  /boot/config.txt or on bookworm and higher /boot/firmware/config.txt:
```
dtoverlay=i2c-gpio,i2c_gpio_sda=27,i2c_gpio_scl=22,i2c_gpio_delay_us=3
dtoverlay=tft_touch
```
This configures an i2c connection with the pins used by IO2CAN and loads the overlay for the touch driver.

- after a reboot
```
sudo reboot now
```
you should have a working touch display.

> [!IMPORTANT]
> If the kernel version changes you will have to build and install the module again:
> ```
> cd btt_tft35/tsc2007
> make clean
> make
> make install
> ```

## Debugging

- check if the module or overlay could be loaded:
```
dmesg
sudo vcdbg log msg
```

## Aditional
#Integrate correct fb:
To get KlipperScreen working edit '99-myfb.conf' and replace with user correct '/dev/FB_' of the display, and move it to '/usr/share/X11/xorg.conf.d/'.
#Example (99-fbdev.conf)
```
Section "Device"
Identifier "LCD"
Driver "fbdev"
Option "fbdev" "/dev/fb0"
Option "Rotate" "180"
Option "SwapbuffersWait" "true"
EndSection
```

#Touch driver calibration and rotation:
```
sudo apt update
sudo apt install xinput-calibrator
```
Find ID of touchscreen
```
DISPLAY=:0 xinput_calibrator --list
```
Start Calibration
```
DISPLAY=:0 xinput_calibrator -v --device <id>
```
Convert to libinput
```
nano libinput_calibrator.sh
```
Paste the following into libinput_calibrator.sh
```
#!/bin/bash

screen_width=$1
screen_height=$2
click_0_X=$3
click_0_Y=$4
click_3_X=$5
click_3_Y=$6

re='^[0-9]+$'
if ! [[ $screen_width =~ $re ]] ; then
  echo "error: screen_width=\"$screen_width\" Not a number" >&2; exit 1
fi
if ! [[ $screen_height =~ $re ]] ; then
  echo "error: screen_height=\"$screen_height\" Not a number" >&2; exit 1
fi
if ! [[ $click_0_X =~ $re ]] ; then
  echo "error: click_0_X=\"$click_0_X\" Not a number" >&2; exit 1
fi
if ! [[ $click_0_Y =~ $re ]] ; then
  echo "error: click_0_Y=\"$click_0_Y\" Not a number" >&2; exit 1
fi
if ! [[ $click_3_X =~ $re ]] ; then
  echo "error: click_3_X=\"$click_3_X\" Not a number" >&2; exit 1
fi
if ! [[ $click_3_Y =~ $re ]] ; then
  echo "error: click_3_Y=\"$click_3_Y\" Not a number" >&2; exit 1
fi

#a = (screen_width * 6 / 8) / (click_3_X - click_0_X)
#c = ((screen_width / 8) - (a * click_0_X)) / screen_width
#e = (screen_height * 6 / 8) / (click_3_Y - click_0_Y)
#f = ((screen_height / 8) - (e * click_0_Y)) / screen_height

a=$(awk "BEGIN { printf(\"%.6f\", ($screen_width * 6 / 8) / ($click_3_X - $click_0_X))}")
c=$(awk "BEGIN { printf(\"%.6f\", (($screen_width / 8) - ($a * $click_0_X)) / $screen_width)}")
e=$(awk "BEGIN { printf(\"%.6f\", ($screen_height * 6 / 8) / ($click_3_Y - $click_0_Y))}")
f=$(awk "BEGIN { printf(\"%.6f\", (($screen_height / 8) - ($e * $click_0_Y)) / $screen_height)}")

CONFIG_OPTION="Option \"CalibrationMatrix\" "
CONFIG_LINE="\"$a 0.000000 $c 0.000000 $e $f 0.000000 0.000000 1.000000\""

echo "${CONFIG_OPTION}${CONFIG_LINE}"
echo ""

CONFIG_OPTION="Option \"CalibrationMatrix\" "
CONFIG="/usr/share/X11/xorg.conf.d/40-libinput.conf"
INPUT_CLASS="Identifier \"libinput touchscreen catchall\""
if [ -e "${CONFIG}" ]; then
    ks_restart=0
    grep -e "^\        ${CONFIG_OPTION}${CONFIG_LINE}" ${CONFIG} > /dev/null
    STATUS=$?
    if [ $STATUS -eq 1 ]; then
        sudo sed -i "/${CONFIG_OPTION}/d" ${CONFIG}
        sudo sed -i "/${INPUT_CLASS}/a\        ${CONFIG_OPTION}${CONFIG_LINE}" ${CONFIG}
        echo "Written to file:"
        echo "    ${CONFIG}"
        echo ""
        ks_restart=1
    fi

    # restart KlipperScreen
    if [ ${ks_restart} -eq 1 ];then
        sudo service KlipperScreen restart
    fi

    echo "run:"
    echo "    DISPLAY=:0 xinput list-props <device>"
    echo "to check if the calibration parameters are effective"
    echo ""
fi
```
Use command below to add executable permissions
```
sudo chmod +x libinput_calibrator.sh
```
Then run libinput_calibrator.sh to convert calibration parameters
```
sudo ./libinput_calibrator.sh <screen width> <screen height> <click_0 X> <click_0 Y> <click_3 X> <click_3 Y>
```
Screen horizontal resolution, TFT35 SPI is 480
Screen vertical resolution, TFT35 SPI is 320
<click_0 X>: The X position of click 0 during the previous step calibration
<click_0 Y>: The Y position of click 0 during the previous step calibration
<click_3 X>: The X position of click 3 during the previous step calibration
<click_3 Y>: The Y position of click 3 during the previous step calibration

Example:
```
sudo ./libinput_calibrator.sh 480 320 61 35 417 281
```
The script will automatically convert and write parameters to the configuration file, and then reset KlipperScreen if installed. You can check whether the configuration is effective through the command
```
DISPLAY=:0 xinput list-prop
```
## Acknowledgements
I like to thank Stenberggg on github for trying this setup on a raspberry pi 5. Fragmon@Crydteam on discord tried this on a CM4 and a Manta board and connect the display to the FCC port. He has also helped a lot in figuring out the correct pins. 
