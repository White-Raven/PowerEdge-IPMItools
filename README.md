# PowerEdge-IPMItools
#### About
Just trying to put as much Dell Poweredge IPMItool related stuff available in one place as possible.

#### Requirements
- iDrac Entreprise (afaik it won't work with express for most, will need help to sort stuff.)
- [IPMItool](https://github.com/ipmitool/ipmitool)
- G11, G12 or G13** Dell Poweredge server (the ones I know most about)

_I was told it is also working on iDrac8 (G13), but that beyond iDrac update 3.30.30.30, Dell has modified/removed the ability to control the fans via IMPI, and there may be other changes I'm not aware of._

#### Before everything

... A little tip. iDrac, in its web interface, truncates the password at 20 characters.
Once you changed your iDrac password from the default root/calvin combo
- if you did it from the web interface, in IPMItool, only use the 20 first characters.
- if you did it from IPMItool... I don't know! That page is open to contributions, and I don't want to risk borking my iDrac to find out... yet! So meanwhile, don't try it?

#### Command synthax and how to make your script less prone to mistakes by repetition.

You can, in your shell script, start by setting variables to make you life easier
```bash
IPMIHOST=192.168.0.42
IPMIUSER=root
IPMIPW=calvin
#Encryption key
IPMIEK=0000000000000000000000000000000000000000

ipmitool -I lanplus -H $IPMIHOST -U $IPMIUSER -P $IPMIPW -y $IPMIEK ##your arguments##
```
As well as dumping that whole monstruosity into an array too for example:
```bash
idrac=(ipmitool -I lanplus -H $IPMIHOST -U $IPMIUSER -P $IPMIPW -y $IPMIEK)

"${idrac[@]}" ##your arguments##
```
Ipmitool can be used in multiple ways, from sending commands to the IPMI of the same machine we're using the tool on, to sending them to the IPMI of an other machine. 

This, and the generation of server you're on will change the -I interface type, between lan and lanplus amongst other ones, or for example if -H will be the actual IPMI host IP, or the nodename in some blade servers. 

**_Side note: you shouldn't ever store credentials in a script. Period. Here it's an example. I suggest you give a look at tools like [/encpass](https://github.com/plyint/encpass.sh) for example._

## Introduction to the forgotten dead-C-scrolls for iDrac that can be handy

Some of them can feel like "eh, no point in having that displayed". Grep is your friend, and a little demo of it:
```bash
ipmitool sdr type temperature
```
returns in my case:
```bash
Inlet Temp | 04h | ok | 7.1 | 20 degrees C
Exhaust Temp | 01h | ok | 7.1 | 39 degrees C
Temp | 0Eh | ok | 3.1 | 41 degrees C
Temp | 0Fh | ok | 3.2 | 40 degrees C
```
BUT
```bash
ipmitool sdr type temperature |grep 0Eh |grep degrees |grep -Po '\d{2}' | tail -1
ipmitool sdr type temperature |grep 0Fh |grep degrees |grep -Po '\d{2}' | tail -1
```
will return:
```bash
41
40
```
Oh. Exploitable data.

## Fan Control - raw
Since we pulled temperatures, let's talk about cooling.
I posted there some time ago my [/PowerEdge-shutup](https://github.com/White-Raven/PowerEdge-shutup) repo, with a basic script to manage server cooling, with potentially lower speed (and noise) than the stock profiles.
Going quickly over these, the commands used are as follow:

```ipmitool raw 0x30 0x30 0x01 0x00``` will stop the server from adjusting fanspeed by itself, no matter the temp, letting you have full manual control

```ipmitool raw 0x30 0x30 0x01 0x01``` will give back to the server the right to automate fan speed, following the profile set in bios/idrac

```ipmitool raw 0x30 0x30 0x02 0xff 0x"hex value 00-64"``` will let you adjust fan speeds.

These are hexadecimal values, 00 is 00, and that 64 is 100. In % in that case.

So ```ipmitool raw 0x30 0x30 0x02 0xff 0x0a``` will set the fan speed to 10% and ```ipmitool raw 0x30 0x30 0x02 0xff 0x0c``` will set the fan speed to 12%, for example.
Once again, you can check my [repo](https://github.com/White-Raven/PowerEdge-shutup) dedicated to fan control to see how this fits into some logic.

## Fan Control - help they are pinned at 100%!!!
Let me guess... you slapped in an unsupported PCIe card. Alright, let's fix that.
First, we must check:
```bash
ipmitool raw 0x30 0xce 0x01 0x16 0x05 0x00 0x00 0x00
```

If it answers ```16 05 00 00 00 05 00 00 00 00```, the Third party PCIe device panic mode is enabled

If it answers ```16 05 00 00 00 05 00 01 00 00```, the Third party PCIe device panic mode is disabled, and then I don't know what's up.


If it is indeed enabled, and you seem to want to shush it, go for
```bash
ipmitool raw 0x30 0xce 0x00 0x16 0x05 0x00 0x00 0x00 0x05 0x00 0x01 0x00 0x00
```

If for some reason you want to enable it back on, just use this:
```bash
ipmitool raw 0x30 0xce 0x00 0x16 0x05 0x00 0x00 0x00 0x05 0x00 0x00 0x00 0x00
```


## Delloem - the commands of Dell... Dell's commands...

Roughtly ```ipmitool delloem [-mx -NPRUEFJTVY] command```

#### Options
<details>
<summary>
<b>[CLICK ME] Table colasped: </b>
</summary>
<p>

Options | Descriptions 
------------ | -------------
```-m 002000 ```| Show FRU for a specific MC (e.g. bus 00, sa 20, lun 00). This could be used for PICMG or ATCA blade systems. The trailing character, if present, indicates SMI addressing if 's', or IPMB addressing if 'i' or not present.
```-x``` | Causes extra debug messages to be displayed.
```-N nodename``` | Nodename or IP address of the remote target system. If a nodename is specified, IPMI LAN interface is used. Otherwise the local system management interface is used.
```-P/-R rmt_pswd``` | Remote password for the nodename given. The default is a null password.
```-U rmt_user``` | Remote username for the nodename given. The default is a null username.
```-E``` | Use the remote password from Environment variable IPMI_PASSWORD.
```-F drv_t``` | Force the driver type to one of the followng: imb, va, open, gnu, landesk, lan, lan2, lan2i, kcs, smb. Note that lan2i means lan2 with intelplus. The default is to detect any available driver type and use it.
```-J``` | Use the specified LanPlus cipher suite (0 thru 14): 0=none/none/none, 1=sha1/none/none, 2=sha1/sha1/none, 3=sha1/sha1/cbc128, 4=sha1/sha1/xrc4_128, 5=sha1/sha1/xrc4_40, 6=md5/none/none, ... 14=md5/md5/xrc4_40. Default is 3.
```-T``` | Use a specified IPMI LAN Authentication Type: 0=None, 1=MD2, 2=MD5, 4=Straight Password, 5=OEM.
```-V``` | Use a specified IPMI LAN privilege level. 1=Callback level, 2=User level, 3=Operator level, 4=Administrator level (default), 5=OEM level.
```-Y``` | Yes, do prompt the user for the IPMI LAN remote password. Alternatives for the password are -E or -P.
</p>
</details>

#### Commands
<details>
<summary>
<b>[CLICK ME] Table colasped: </b>
</summary>
<p>

Commands | Descriptions 
------------ | -------------
```mac list``` | Lists the MAC address of LOMs
```mac get <NIC number>``` | Shows the MAC address of specified LOM. 0-7 System LOM, 8- DRAC/iDRAC.
```lan set <Mode>``` | Sets the NIC Selection Mode (dedicated, shared, shared with failover lom2, shared with Failover all loms).
```lan get``` | Returns the current NIC Selection Mode (dedicated, shared, shared with failover lom2, shared with Failover all loms).
```lan get active``` | Returns the current active NIC (dedicated, LOM1, LOM2, LOM3, LOM4).
```powermonitor``` | Shows power tracking statistics
```powermonitor clear cumulativepower``` | Reset cumulative power reading
```powermonitor clear peakpower``` | Reset peak power reading
```powermonitor powerconsumption``` | Displays power consumption in <watt|btuphr>
```powermonitor powerconsumptionhistory <watt\|btuphr>``` | Displays power consumption history
```powermonitor getpowerbudget``` | Displays power cap in ```<watt\|btuphr>```
```powermonitor setpowerbudget <val> <watt\|btuphr\|percent>``` | Allows user to set the power cap in ```<watt\|BTU/hr\|percentage>```
```powermonitor enablepowercap``` |To enable set power cap
```powermonitor disablepowercap``` | To disable set power cap
```windbg start``` | Starts the windbg session (Cold Reset & SOL Activation)
```windbg end``` | Ends the windbg session (SOL Deactivation)
```vFlash info Card``` | Shows Extended SD Card information
</p>
</details>

For example :
  
```ipmitool delloem powermonitor```
returns in my case:
```Power Tracking Statistics
Statistic : Cumulative Energy Consumption
Start Time : 02/20/20 10:03:52 CETFinish Time : 07/25/21 09:30:16 CESTReading : 1719.3 kWh

Statistic : System Peak Power
Start Time : 02/20/20 10:03:52 CETPeak Time : 07/13/21 05:45:27 CESTPeak Reading : 370 W

Statistic : System Peak Amperage
Start Time : 02/20/20 10:03:52 CETPeak Time : 07/13/21 05:45:27 CESTPeak Reading : 4.5 A
```
Just have fun with grep.

## IPMI commands
  ```ipmitool #command```
<details>
<summary>
<b>[CLICK ME] Table colasped: </b>
</summary>
<p>

|Commands | Descriptions |
| --------------------------------- |-------------|
|``` mc info``` | To check firmware version |
|``` mc reset [ warm \| cold ]```| To reset the management controller |
|``` fru print``` | Show field-replaceable-unit details |
|``` sdr list``` <br> ``` sdr type list``` <br> ``` sdr type Temperature``` <br> ``` sdr type Fan``` <br> ``` sdr type ‘Power Supply’``` | Show sensor output |
|``` chassis status``` | Well... status |
|``` chassis identify []``` | turn on front panel identify light (default 15s) |
|``` [chassis] power soft``` | initiate a soft-shutdown via acpi |
|``` [chassis] power cycle``` | issue a hard power off, wait 1s, power on |
|``` [chassis] power off``` | issue a hard power off |
|``` [chassis] power on``` | issue a hard power on |
|``` [chassis] power reset``` | issue a hard reset |
|``` chassis bootdev pxe``` <br> ``` chassis bootdev cdrom``` <br> ``` chassis bootdev bios``` | Modify boot device for next reboot|
|```ipmitool lan print 1``` <br> ```ipmitool lan set 1 ipsrc [ static \| dhcp ]``` <br> ```ipmitool lan set 1 ipaddr {YOUR DESIRED IP}``` <br> ```ipmitool lan set 1 netmask {YOUR NETMASK}``` <br> ```ipmitool lan set 1 defgw ipaddr 10.0.1.1``` | Display/configure lan settings if you're not connecting to the IPMI from an other machine, <br> but configuring it locally. <br> I'm not sure how it behaves to reconfigure like that an IPMI <br>to which you're actively connected through something else than the web interface. |
|``` bmc reset cold``` | Hanging BMC, hardreset |

</p>
</details>

- You can also check the [man page](https://linux.die.net/man/1/ipmitool) of IPMI tool for more in depth info.

## The big guns - sol activate
In your BIOS, you can enable SOL. Serial over Lan. A potentially powerfull tool for administration.

You can enable SOL functionality in the BIOS utility by first rebooting the server and pressing F2 to launch the utility. 

In the Serial Communication options, they should then set the Serial Communication setting to On with Console Redirection via COM2, the External Serial Connector setting to Remote Access Device, the Failsafe Baud Rate setting to any suitable value (the BIOS attempts to determine this value automatically, and uses this baud rate only if that attempt fails).

```ipmitool  sol activate``` is basically all you need then to launch it.

Only one SOL session can be active at a time. 
<details>
<summary>
<b>[CLICK ME] Here the cheatsheet of what you can use as various escape sequences within a SOL session: </b>
</summary>
<p>

Keyboard mapping for console redirection or session task | Escape sequence 
------------ | -------------
Terminate connection | ~+.
Suspend IPMItool | ~+^+Z
Send break | ~+B
Print escape sequence help | ~+?
F1 | Esc+1
F2 | Esc+2
F3 | Esc+3
F9 | Esc+4
F10 | Esc+0
F11 | Esc+!
F12 | Esc+@
Home | Esc+h
End | Esc+k
Insert | Esc++
Delete | Esc+-
Page up | Esc+?
Page down | Esc+/
Ctrl+M | Esc+Ctrl+M
Ctrl+H | Esc+Ctrl+H
Ctrl+I | Esc+Ctrl+I
Ctrl+J | Esc+Ctrl+J
Alt+x (where x is any letter) | Esc+X+x (where x is any letter, and X is the uppercase of that letter)
Ctrl+Alt+Del | Esc+R+Esc+r+Esc+R
</p>
</details>

The escape sequence ~+. terminates the session and resets the terminal settings. However, if SOL mode exits unintentionally and the BMC must be reset, you can also terminate the current session from another console using the following command:
```bash
ipmitool  sol deactivate
```

<details>
<summary>
<b>[CLICK ME] inittab and getty no longer works in RHEL7, so have fun with some config for systemd. </b>
</summary>
<p>

```bash
## /etc/systemd/system/mgetty.service ::

[Unit]
Description=Smart Modem Getty(mgetty)
Documentation=man:mgetty(8)
Requires=systemd-udev-settle.service
After=systemd-udev-settle.service

[Service]
Type=simple
ExecStart=/sbin/mgetty -r -s 115200 /dev/ttyS1
# -r is for direct connection (ie null modem)
Restart=always
PIDFile=/var/run/mgetty.pid.ttyS1

[Install]
WantedBy=basic.target multi-user.target
```
and 

```bash
ls -Z /usr/sbin/mgetty
ls -Z /var/log/mgetty.*
chcon --type getty_exec_t /var/log/mgetty.*.log
### still need more tweaking, but don't really prevent mgetty from creating a login session.

systemctl start  mgetty.service
systemctl enable mgetty.service
# enable will create sym links in /etc/systemd/system/basic.target.wants and multi-user.target.wants

# create ipmi user #3 to have access
ipmitool user list
ipmitool user set password 3 NewIpmiPwForUserNum3	# if need to change password
ipmitool user enable 3
ipmitool channel setaccess 1 3 ipmi=on privilege=4
ipmitool user priv 3 4 1
```
</p>
</details>


## Blade stuff
  
#### Fishing for Dell Chassis ServiceTag from a node using hexcode and node programming

Dell C6320 and similar blade servers, their chassis has its own service tag.
Just a bit of hex code in raw can retrieve them, though they would have to be converted back
Optional : xxd -r reverse hex conversion to ascii, avail from vim-common rpm

Example: Dell C6320 vintage 2017
```ipmitool  raw 0x30 0xc8 0x01 0x00 0x0b 0x00 0x00 0x00 | xxd -r```

It's possible it gives nothing, for example the C6220II should use the same commands as the C6320, but if it's an too much older blade system, then it might not work.
The C6100 and C6220 for example don't have their own unique chassis ServiceTag to query.
To at least try to query the Set FCB values, run ALL of the following commands in this order
(It generate and execute some sort of reservation, each command should produce some garbagio, till the last one should yield a serial number for the chassis)

```ipmitool raw 0x30 0xC8 0x01 0x00 0x02 0x00 0x00 0x00```

```ipmitool raw 0x30 0xC8 0x01 0x00 0x02 0x00 0x00 0x00 0x00```

```ipmitool raw 0x30 0xC8 0x01 0x00 0x02 0x00 0x00 0x00 0x01```

```ipmitool raw 0x30 0xC8 0x01 0x00 0x0c 0x00 0x02 0x00 0x00 0x00 | xxd -r ```

#### Programming a Dell Chassis from a node using some raw commands
Some chassis have their ServiceTag programmed from factory, others are not. 
For inventory purpose, can program-in the service tag so that they can be queried. 
For example there we're gonna program the Service Tag of "bjhkqd2" to the chassis via one of its node.
bjhkq69 in hex is ```62 6a 68 6b 71 36 39``` which becomes ```0x62 0x6a 0x68 0x6b 0x71 0x36 0x39```
```bash
ipmitool raw 0x30 0xC8 0x00 0x00 0x0B 0x00 0x00 0x00 0x0B 0x00 0x11 0x0A 0x62 0x6a 0x68 0x6b 0x71 0x36 0x39
## commit
ipmitool raw 0x30 0xC8 0x01 0x00 0x02 0x00 0x00 0x00
```


## The RAW command disclaimer.

You can guess I didn't pulled the raw commands out my rear IO, nor did I brutforced them. I respect my hardware... a bit.

If you wanna help in uncovering more commands that work, swallow [that](https://www.intel.com/content/dam/www/public/us/en/documents/specification-updates/ipmi-intelligent-platform-mgt-interface-spec-2nd-gen-v2-0-spec-update.pdf) and here's the model:

For example, generating platform event messages:

```ipmitool raw netfn cmd evmrev sensortype sensorid eventdir/eventtype eventdata``` where:
- (netfn): Table 5-1 of the IPMI 2.0 specification list
-  (cmd): Appendix G of the IPMI 2.0 specification list
-  (evmrev): the event message revision is 04h in IPMI 2.0 specification list
-  (sensortype): Table 42-3 of the IPMI 2.0 specification list
-  (sensorid): The sensor ID value for the processors for example can be obtained using the sensor command with a bit of grep: ```ipmitool -v -I lanplus -H $IPMIHOST -U $IPMIUSER -P $IPMIPW -y $IPMIEK sensor | grep -B 4 'Processor' | grep 'Sensor ID' | grep -Po '0x\d{2}'```
-  (eventdir/eventtype): Table 29-5 of the IPMI 2.0 specification lists the event direction for an assertion event as 0, and Table 42-1 lists the event type for a sensor-specific event (such as “processor disabled”) as 6Fh. Combining the event direction, 0h (0), with the event type, 6Fh (1101111), produces a final binary value of 01101111, which is equivalent to 0x6F.
-  (eventdata): Table 42-3 of the IPMI 2.0 specification lists

Which lead to platform event messages generating commands such as:

Platform event message | Raw command 
------------ | -------------
Processor Disabled | ```ipmitool raw 0x04 0x02 0x04 0x07 0x61 0x6F 0x08 00 00```
Over temperature | ```ipmitool raw 0x04 0x02 0x04 0x01 0x01 0x01 0x09 00 00```
Over voltage | ```ipmitool raw 0x04 0x02 0x04 0x02 0x11 0x01 0x09 00 00```
Chassis intrusion | ```ipmitool raw 0x04 0x02 0x04 0x05 0x52 0x6f 0x00 00 00```
Fan redundancy lost | ```ipmitool raw 0x04 0x02 0x04 0x04 0x30 0x0b 0x01 00 00```
Power redundancy lost | ```ipmitool raw 0x04 0x02 0x04 0x08 0x44 0x0b 0x01 00 00```
Power supply failure | ```ipmitool raw 0x04 0x02 0x04 0x08 0x44 0x6f 0x01 00 00```
Power supply lost | ```ipmitool raw 0x04 0x02 0x04 0x08 0x44 0x6f 0x03 00 00```
Memory redundancy lost | ```ipmitool raw 0x04 0x02 0x04 0x0c 0x13 0x0b 0x01 00 00```
System event log full | ```ipmitool raw 0x04 0x02 0x04 0x10 0x51 0x6f 0x04 00 00```

Whiiiich can't be run as is because the sensor IDs are platform specific. 

Once the Platform Event Trap events are generated, you can verify them using the command ipmitool -I open sel list.
  

<details>
<summary>  
<b>[Click me maybe]</b>
</summary>
<p>

# NIGHTMARE NIGHTMARE NIGHTMARE NIGHTMARE NIGHTMARE NIGHTMARE NIGHTMARE NIGHTMARE NIGHTMARE NIGHTMARE

</p>
</details>
