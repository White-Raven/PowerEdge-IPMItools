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
Start Time : 02/20/20 10:03:52 CETPeak Time : 07/13/21 05:45:27 CESTPeak Reading : 4.5 A```








----------------

**_I was told it is also working on iDrac8 (G13), but that beyond iDrac update 3.30.30.30, Dell has modified/removed the ability to control the fans via IMPI, and there may be other changes I'm not aware of._
