# PowerEdge-IPMItools

## Requirements
- iDrac Entreprise (afaik it won't work with express for most, will need help to sort stuff.)
- [IPMItool](https://github.com/ipmitool/ipmitool)
- G11, G12 or G13** Dell Poweredge server (the ones I know most about)

## Before everything

... A little tip. iDrac, in its web interface, truncates the password at 20 characters.
Once you changed your iDrac password from the default root/calvin combo
- if you did it from the web interface, in IPMItool, only use the 20 first characters.
- if you did it from IPMItool... I don't know! That page is open to contributions, and I don't want to risk borking my iDrac to find out... yet! So meanwhile, don't try it?

## Command synthax and how to make your script less prone to mistakes by repetition.

You can, in your shell script, start by setting variables to make you life easier
```
IPMIHOST=192.168.0.69
IPMIUSER=root
IPMIPW=calvin
#Encryption key
IPMIEK=0000000000000000000000000000000000000000

ipmitool -I lanplus -H $IPMIHOST -U $IPMIUSER -P $IPMIPW -y $IPMIEK ##your arguments##
```
As well as dumping that whole monstruosity into an array too for example:
```
idrac=(ipmitool -I lanplus -H $IPMIHOST -U $IPMIUSER -P $IPMIPW -y $IPMIEK)

"${idrac[@]}" ##your arguments##
```

## Various Dell commands that can be handy

#### Muh powerbill - delloem powermonitor
```"${idrac[@]}" delloem powermonitor```
returns in my case:
```Power Tracking Statistics
Statistic : Cumulative Energy Consumption
Start Time : 02/20/20 10:03:52 CETFinish Time : 07/25/21 09:30:16 CESTReading : 1719.3 kWh

Statistic : System Peak Power
Start Time : 02/20/20 10:03:52 CETPeak Time : 07/13/21 05:45:27 CESTPeak Reading : 370 W

Statistic : System Peak Amperage
Start Time : 02/20/20 10:03:52 CETPeak Time : 07/13/21 05:45:27 CESTPeak Reading : 4.5 A
```
#### The big guns - sol activate
In your BIOS, you can enable SOL. Serial over Lan. A potentially powerfull tool for administration.

You can enable SOL functionality in the BIOS utility by first rebooting the server and pressing F2 to launch the utility. 

In the Serial Communication options, they should then set the Serial Communication setting to On with Console Redirection via COM2, the External Serial Connector setting to Remote Access Device, the Failsafe Baud Rate setting to any suitable value (the BIOS attempts to determine this value automatically, and uses this baud rate only if that attempt fails).

```"${idrac[@]}"  sol activate``` is basically all you need then to launch it.

Only one SOL session can be active at a time. Here the cheatsheet of what you can use as various escape sequences within a SOL session:
Keyboard mapping for console redirection or session task | Escape sequence |
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





----------------

**_I was told it is also working on iDrac8 (G13), but that beyond iDrac update 3.30.30.30, Dell has modified/removed the ability to control the fans via IMPI, and there may be other changes I'm not aware of._
