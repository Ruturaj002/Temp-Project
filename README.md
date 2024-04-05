Code organization as follows

- Helper script = contains code for setting level of 10 segment display
- batch_controller.py = contains script for managing communication and handling with batch controller
- main.py = main code which runs forever



Follow the steps to configure 4G module

- configure 4g module to run in RNDIS mode. Enter following command in command prompt
```
screen /dev/ttyUSB2
```
- Enter following AT command to enable RNDIS mode

```
AT+CUSBPIDSWITCH=9011,1,1
```

- Change host name of beaglebone black

```
sudo nano /etc/hosts
```
open file and replace beaglebone with following
naming format = Janyutech-Edosing-XXX


```
sudo nano /etc/hostname
```  

- Setup service for automatic running on boot
- Create a service named edosing.service
```
sudo nano /lib/systemd/system/edosing.service
```
- 
```
[Unit]
Description=Edosing Logic
After=network.target
StartLimitIntervalSec=10
StartLimitBurst=3

[Service]
ExecStart=/usr/bin/python3 /home/debian/main.py 
WorkingDirectory=/home/debian
Restart=on-failure
RestartSec=1s


[Install]
WantedBy=multi-user.target

```


- add UART overlay

```
sudo nano /boot/uEnv.txt
```

add following text below enable_overlay_=1


```
# UART 1
uboot_overlay_addr0=/lib/firmware/BB-UART1-00A0.dtbo
# UART 2
uboot_overlay_addr1=/lib/firmware/BB-UART2-00A0.dtbo
# UART 4
uboot_overlay_addr2=/lib/firmware/BB-UART4-00A0.dtbo
# UART 5
uboot_overlay_addr3=/lib/firmware/BB-UART5-00A0.dtbo

# P8_28
uboot_overlay_addr4=/lib/firmware/BB-P8_28-4G-00A0.dtbo


```
- Update nodered to latest version
```
sudo apt update
sudo apt upgrade nodejs bb-node-red-installer
```

- Enable node-red service to start on boot
```
sudo systemctl enable nodered.service
```

- Install pymodbus
```
pip3 install -U pymodbus
```
- Install TB Mqtt Client
```
pip3 install tb-mqtt-client==1.4
```
- Enable Edosing python service running
```
sudo systemctl enable edosing.service
```

- clone repository as follows
```
git clone https://github.com/beagleboard/bb.org-overlays

```

- write a device tree for P8_28 for control of simcom reset pin

```
sudo nano bb.org-overlays/src/arm/BB-P8_28-4G-00A0.dts
```

 

```
/*
 * Copyright (C) 2020 Robert Nelson <robertcnelson@gmail.com>
 *
 * This program is free software; you can redistribute it and/or modify
 * it under the terms of the GNU General Public License version 2 as
 * published by the Free Software Foundation.
 */

/dts-v1/;
/plugin/;

#include <dt-bindings/board/am335x-bbw-bbb-base.h>
#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/pinctrl/am33xx.h>

/ {
	/*
	 * Helper to show loaded overlays under: /proc/device-tree/chosen/overlays/
	 */
	fragment@0 {
		target-path="/";
		__overlay__ {

			chosen {
				overlays {
					BB-P8_28-4G-00A0 = __TIMESTAMP__;
				};
			};
		};
	};

	/*
	 * Free up the pins used by the cape from the pinmux helpers.
	 */
	fragment@1 {
		target = <&ocp>;
		__overlay__ {
			P8_28_pinmux { status = "disabled"; };
		};
	};

	fragment@2 {
		target = <&am33xx_pinmux>;
		__overlay__ {

			bb_gpio_led_pins: pinmux_bb_gpio_led_pins {
				pinctrl-single,pins = <
					BONE_P8_28 (PIN_OUTPUT_PULLUP | MUX_MODE7)	
				>;
			};
		};
	};

	fragment@3 {
		target-path="/";
		__overlay__ {

			leds {
				pinctrl-names = "default";
				pinctrl-0 = <&bb_gpio_led_pins>;

				compatible = "gpio-leds";

				P8_28 {
					label = "P8_28";
					gpios = <&gpio2 24 GPIO_ACTIVE_HIGH>;
					default-state = "on";
				};
			};
		};
	};
};
```


```
cd bb.org-overlays
```

```
make
sudo make install
```




- node-red flow to reset 4G if device goes offline

- install isonline node from manage pallete

```
[{"id":"a18083a.d2f6a8","type":"tab","label":"Flow 1","disabled":false,"info":""},{"id":"997174f3.b2f588","type":"is online","z":"a18083a.d2f6a8","name":"","url":"","action":"0","x":460,"y":120,"wires":[["79a87dfa.e677d4"]]},{"id":"3699a48b.19138c","type":"inject","z":"a18083a.d2f6a8","name":"","props":[{"p":"payload"},{"p":"topic","vt":"str"}],"repeat":"7200","crontab":"","once":false,"onceDelay":0.1,"topic":"","payload":"","payloadType":"date","x":240,"y":100,"wires":[["997174f3.b2f588"]]},{"id":"b600f634.1ae268","type":"debug","z":"a18083a.d2f6a8","name":"","active":true,"tosidebar":true,"console":false,"tostatus":false,"complete":"false","statusVal":"","statusType":"auto","x":930,"y":200,"wires":[]},{"id":"79a87dfa.e677d4","type":"switch","z":"a18083a.d2f6a8","name":"","property":"payload","propertyType":"msg","rules":[{"t":"false"},{"t":"true"}],"checkall":"true","repair":false,"outputs":2,"x":700,"y":140,"wires":[["be37db06.590a88"],["b600f634.1ae268"]]},{"id":"be37db06.590a88","type":"exec","z":"a18083a.d2f6a8","command":"/home/debian/reset_4G.sh","addpay":"","append":"","useSpawn":"false","timer":"","oldrc":false,"name":"reset 4G","x":910,"y":60,"wires":[["604f34e9.bf4f5c"],[],[]]},{"id":"604f34e9.bf4f5c","type":"debug","z":"a18083a.d2f6a8","name":"","active":true,"tosidebar":true,"console":false,"tostatus":false,"complete":"false","statusVal":"","statusType":"auto","x":1200,"y":60,"wires":[]}]
```

- shell script to reset 4G module

```
sudo nano reset_4G.sh
```
  
```
echo 1 > /sys/class/leds/P8_28/brightness
sleep 1s
echo 0 > /sys/class/leds/P8_28/brightness
sleep 1s
echo 1 > /sys/class/leds/P8_28/brightness
```

```
sudo chmod +777 reset_4G.sh
```


```


