LED Controller of UGREEN's DX4600 Pro NAS
==

UGREEN's DX4600 Pro is a four-bay NAS with a built-in system based on OpenWRT called `UGOS`. It can install Debian or other open-source NAS systems, but the issue is that the installed non-UGOS system does not have drivers for the six LED lights on the front panel (indicating power, network card, and four hard drives). By default, only the power indicator light blinks, and other indicator lights are off.

This repository describes the control logic of UGOS for these LED lights and provides a command-line tool and a kernel module to control them. For the process of understanding this control logic, please refer to [my blog (in Chinese)](https://blog.miskcoo.com/2024/05/ugreen-dx4600-pro-led-controller).

**WARNING:** Only tested on the following devices. I guess that it works for all DX4600 series. For other devices, please follow the [Preparation](#Preparation) section to check if the protocol is compatible, and run `./ugreen_leds_cli all` to see which LEDs are supported by this tool.

- [x] UGREEN DX4600 Pro
- [x] UGREEN DXP4800 Plus (reported [here](https://gist.github.com/Kerryliu/c380bb6b3b69be5671105fc23e19b7e8))
- [x] UGREEN DXP6800 Pro (reported in [#7](https://github.com/miskcoo/ugreen_dx4600_leds_controller/issues/7))
- [x] UGREEN DXP8800 Plus (see [this repo](https://github.com/meyergru/ugreen_dxp8800_leds_controller) and [#1](https://github.com/miskcoo/ugreen_dx4600_leds_controller/issues/1))
- [ ] UGREEN DXP480T Plus (**NO**, but the protocol has been understood, see [#6](https://github.com/miskcoo/ugreen_dx4600_leds_controller/issues/6#issuecomment-2156807225))

**I am not sure whether this is compatible with other devices. If you have tested it in other devices, please feel free to update the list above.**

Below is an example:

![](https://blog.miskcoo.com/assets/images/dx4600-pro-leds.gif)

It can be achieved by the following commands:
```bash
ugreen_leds_cli all -off -status
ugreen_leds_cli power  -color 255 0 255 -blink 400 600 
sleep 0.1
ugreen_leds_cli netdev -color 255 0 0   -blink 400 600
sleep 0.1
ugreen_leds_cli disk1  -color 255 255 0 -blink 400 600
sleep 0.1
ugreen_leds_cli disk2  -color 0 255 0   -blink 400 600
sleep 0.1
ugreen_leds_cli disk3  -color 0 255 255 -blink 400 600
sleep 0.1
ugreen_leds_cli disk4  -color 0 0 255   -blink 400 600
```

## Preparation

We communicate with the control chip of the LED via I2C, corresponding to the device with address `0x3a` on *SMBus I801 adapter*. Before proceeding, we need to load the `i2c-dev` module and install the `i2c-tools` tool.

```
$ apt install -y i2c-tools
$ modprobe -v i2c-dev
```

Now, we can check if the device located at address `0x3a` of *SMBus I801 adapter* is visible.

```
$ i2cdetect -l
i2c-0   i2c             Synopsys DesignWare I2C adapter         I2C adapter
i2c-1   smbus           SMBus I801 adapter at efa0              SMBus adapter
i2c-2   i2c             Synopsys DesignWare I2C adapter         I2C adapter

$ i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         08 -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: 30 -- -- -- -- 35 UU UU -- -- 3a -- -- -- -- --
40: -- -- -- -- 44 -- -- -- -- -- -- -- -- -- -- --
50: UU -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

## Build & Usage

There are three methods to control the LEDs: stateless, userspace daemon, kernel module.

- **stateless**: the simplest way is to directly build and run the `ugreen_leds_cli` tool. It will find and communicate with `/dev/i2c-X`, and does not save the current state of LEDs. Therefore, we are unable to implement the stateful functions like blink an LED once, unless manually maintaining the states.
- **userspace daemon**: it also communicates with `/dev/i2c-X`, but will maintain the LEDs' states and create a socket `/tmp/led-ugreen.socket`. The tool `ugreen_leds_cli` will communicate with the daemon via the socket. This method allows a similar function like the `oneshot` trigger in the kernel module. You can run `ugreen_leds_cli disk1 -oneshot 100 100` to setup the trigger and run `ugreen_leds_cli disk1 -shot` to blink `disk1` once. To use this daemon, you should first run `ugreen_daemon` (it won't exit, keep it running) and then use `ugreen_leds_cli` as before. The `ugreen-diskiomon` script requires this control method or the kernel module below.
  - _TL;DR_: copy `ugreen_daemon` and `ugreen_leds_cli` to `/usr/bin`; keep `ugreen_daemon` running; then keep `scripts/ugreen-diskiomon` running.
- **kernel module** (suggested): it will register LEDs in `/sys/class/leds`, therefore, any triggers supported by the kernel are also usable.

The `ugreen_leds_cli` tool is suitable for all three methods. It will first check whether `/tmp/led-ugreen.socket` exists, and then check whether `/sys/class/leds` contains corresponding LEDs. If both of them do not exist, then it will try to use the stateless method.

### The Command-line Tool

Use `cd cli && make` to build the command-line tool, and `ugreen_leds_cli` to modify the LED states (requires root permissions).

```
Usage: ugreen_leds_cli  [LED-NAME...] [-on] [-off] [-(blink|breath) T_ON T_OFF]
                    [-color R G B] [-brightness BRIGHTNESS] [-status]

       LED_NAME:    separated by white space, possible values are
                    { power, netdev, disk[1-8], all }.
       -on / -off:  turn on / off corresponding LEDs.
       -blink / -breath / -oneshot:  set LED to the blink / breath / oneshot mode. 
                    This mode keeps the LED on for T_ON millseconds and then
                    keeps it off for T_OFF millseconds.
                    T_ON and T_OFF should belong to [0, 65535].
       -color:      set the color of corresponding LEDs.
                    R, G and B should belong to [0, 255].
       -brightness: set the brightness of corresponding LEDs.
                    BRIGHTNESS should belong to [0, 255].
       -status:     display the status of corresponding LEDs.
       -shot:       emit a blink cycle of corresponding LEDs.
```

Below is an example:

```bash
# turn on all LEDs
ugreen_leds_cli all -on

# query LEDs' status
ugreen_leds_cli all -status

# turn on the power indicator,
# and then set its color to blue,
# and then set its brightness to 128 / 256,
# and finally display its status
ugreen_leds_cli power -on -color 0 0 255 -brightness 128 -status
```

### The Kernel Module

There are three methods to install the module:

- Run `cd kmod && make` to build the kernel module, and then load it with `sudo insmod led-ugreen.ko`.

- Alternatively, you can install it with dkms:

  ```bash
  cp -r kmod /usr/src/led-ugreen-0.1
  dkms add -m led-ugreen -v 0.1
  dkms build -m led-ugreen -v 0.1 && dkms install -m led-ugreen -v 0.1
  ```

- You can also directly install the package [here](https://github.com/miskcoo/ugreen_dx4600_leds_controller/releases).

After loading the `led-ugreen` module, you need to run `scripts/ugreen-probe-leds`, and you can see LEDs in `/sys/class/leds`.

Below is an example of setting color, brightness, and blink of the `power` LED:

```bash
echo 255 > /sys/class/leds/power/brightness    # non-zero brightness turns it on
echo "255 0 0" > /sys/class/leds/power/color   # set the color to RGB(255, 0, 0)
echo "blink 100 100" > /sys/class/leds/power/blink_type  # blink at 10Hz
```

To blink the `netdev` LED when an NIC is active, you can use the `ledtrig-netdev` module (see `scripts/ugreen-netdevmon`):

```bash
led="netdev"
modprobe ledtrig-netdev
echo netdev > /sys/class/leds/$led/trigger
echo enp2s0 > /sys/class/leds/$led/device_name
echo 1 > /sys/class/leds/$led/link
echo 1 > /sys/class/leds/$led/tx
echo 1 > /sys/class/leds/$led/rx
echo 100 > /sys/class/leds/$led/interval
```

To blink the `disk` LED when a block device is active, you can use the `ledtrig-oneshot` module and monitor the changes of`/sys/block/sda/stat` (see `scripts/ugreen-diskiomon` for an example). If you are using zfs, you can combine this script with that provided in [#1](https://github.com/miskcoo/ugreen_dx4600_leds_controller/issues/1) to change the LED's color when a disk drive failure occurs. To see how to map the disk LEDs to correct disk slots, please read the [Disk Mapping](#disk-mapping) section.

#### Start at Boot (for Debian 12)

- Edit `/etc/modules-load.d/ugreen-led.conf` and add the following lines:
```
i2c-dev
led-ugreen
ledtrig-oneshot
ledtrig-netdev
```

- Install the `smartctl` tool: `apt install smartmontools`

- Install the kernel module by one of the three methods mentioned above. For example, directly install [the deb package](https://github.com/miskcoo/ugreen_dx4600_leds_controller/releases).

- Copy files in the `scripts` directory: 
```bash
scripts=(ugreen-diskiomon ugreen-netdevmon ugreen-probe-leds)
for f in ${scripts[@]}; do
    chmod +x "scripts/$f"
    cp "scripts/$f" /usr/bin
done

cp scripts/*.service /etc/systemd/system/

systemctl daemon-reload

# change enp2s0 to the network device you want to monitor
systemctl start ugreen-ledmon@enp2s0 

# if you confirm that everything works well, 
# run the command below to make the service start at boot
systemctl enable ugreen-ledmon@enp2s0 
```

## Disk Mapping

To make the disk LEDs useful, we should map the disk LEDs to correct disk slots. First of all, we should highlight that using `/dev/sdX` is never a smart idea, as it may change at every boot. In the script `ugreen-diskiomon` we provide two mapping methods: **by HCTL** and **by serial**. 

The HCTL mapping depends on how the SATA controllers are connected to the PCIe bus and the disk slots. To check the HCTL order, you can run the following command, and check the serial of your disks:

```bash
# lsblk -S -x hctl -o name,hctl,serial
NAME HCTL       SERIAL
sda  0:0:0:0    XXKEREXX
sdc  1:0:0:0    XXKG2BXX
sdb  2:0:0:0    XXGMU6XX
sdd  3:0:0:0    XXKJEZXX
sde  4:0:0:0    XXKJHBXX
sdf  5:0:0:0    XXGT2ZXX
sdg  6:0:0:0    XXKH3SXX
sdh  7:0:0:0    XXJDB1XX
```

As far as we know, the mapping between HCTL and the disk serial are stable at each boot (see [#4](https://github.com/miskcoo/ugreen_dx4600_leds_controller/pull/4) and [#9](https://github.com/miskcoo/ugreen_dx4600_leds_controller/issues/9)). However, it has been reported that the exact order is model-dependent (see [#9](https://github.com/miskcoo/ugreen_dx4600_leds_controller/issues/9)). In DX4600 Pro and DXP8800 Plus, the mapping is `X:0:0:0 -> diskX`, but In the DXP6800 Pro, `0:0:0:0` and  `1:0:0:0` are mapped to `disk5` and `disk6`, and `2:0:0:0` to `6:0:0:0` are mapped to `disk1` to `disk4`, so you should change the array `hctl_map` in `ugreen-diskiomon` accordingly.

## Communication Protocols

The IDs for the six LED lights on the front panel of the chassis are as follows: 

- power indicator light = 0;
- network device indicator light = 1;
- four hard drive indicator lights = 2 - 5.

### Query status

Reading 11 bytes from the address `0x81 + LED_ID` allows us to obtain the current status of the corresponding LED. The meaning of these 11 bytes is as follows:

| Address | Meaning of Corresponding Data |
|---------|--------------------------------|
| 0x00    | LED status: 0 - off, 1 - on, 2 - blink, 3 - breath |
| 0x01    | LED brightness |
| 0x02    | LED color (Red component in RGB) |
| 0x03    | LED color (Green component in RGB) |
| 0x04    | LED color (Blue component in RGB) |
| 0x05    | Milliseconds needed to complete one blink / breath cycle (high 8 bits) |
| 0x06    | Milliseconds needed to complete one blink / breath cycle (low 8 bits) |
| 0x07    | Milliseconds the LED is on during one blink / breath cycle (high 8 bits) |
| 0x08    | Milliseconds the LED is on during one blink / breath cycle (low 8 bits) |
| 0x09    | Checksum of data in the range 0x00 - 0x08 (high 8 bits) |
| 0x0a    | Checksum of data in the range 0x00 - 0x08 (low 8 bits) |

The checksum is a 16-bit value obtained by summing all the data at the corresponding positions as unsigned integers.

We can directly use `i2cget` to read from the relevant registers. For example, below is the status of the power indicator light (purple, blinking once per second, lit for 40% of the time, with a brightness of 180/256):

```
$ i2cget -y 0x01 0x3a 0x81 i 0x0b
0x02 0xb4 0xff 0x00 0xff 0x03 0xe8 0x01 0x90 0x04 0x30
```

### Change status

By writing 12 bytes to the address `0x00 + LED_ID`, we can modify the current status of the corresponding LED. The meaning of these 12 bytes is as follows:

| Address | Meaning of Corresponding Data |
|---------|--------------------------------|
| 0x00    | LED ID |
| 0x01    | Constant: 0xa0 |
| 0x02    | Constant: 0x01 |
| 0x03    | Constant: 0x00 |
| 0x04    | Constant: 0x00 |
| 0x05    | If the value is 1, it indicates modifying brightness. <br/>If the value is 2, it indicates modifying color. <br/>If the value is 3, it indicates setting the on/off state.<br/>If the value is 4 / 5, it indicates setting the blink / breath state. |
| 0x06    | First parameter |
| 0x07    | Second parameter |
| 0x08    | Third parameter |
| 0x09    | Fourth parameter |
| 0x0a    | Checksum of data in the range 0x01 - 0x09 (high 8 bits) |
| 0x0b    | Checksum of data in the range 0x01 - 0x09 (low 8 bits) |

For the four different modification types at address 0x05:

- If we need to modify brightness, the first parameter contains brightness information.
- If we need to modify color, the first three parameters represent RGB information.
- If we need to toggle the on/off state, the first parameter is either 0 or 1, representing off or on, respectively.
- If we need to set the blink / breath state, the first two parameters together form a 16-bit unsigned integer in big-endian order, representing the number of milliseconds needed to complete one blink / breath cycle. The next two parameters, also in big-endian order, represent the number of milliseconds the LED is on during one blink / breath cycle.

Below is an example for turning off and on the power indicator light using `i2cset`:

```
$ i2cset -y 0x01 0x3a 0x00  0x00 0xa0 0x01 0x00 0x00 0x03 0x01 0x00 0x00 0x00 0x00 0xa5 i    # turn off power LED
$ i2cset -y 0x01 0x3a 0x00  0x00 0xa0 0x01 0x00 0x00 0x03 0x00 0x00 0x00 0x00 0x00 0xa4 i    # turn on power LED
```

## Acknowledgement

ChatGPT, [this V2EX post](https://fast.v2ex.com/t/991429), Ghidra 
