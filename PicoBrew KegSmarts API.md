# KegSmarts APIs

The KegSmarts uses HTTP in requests and responses.  It appears all data transfers are initiated via a GET request from the device to PicoBrew's server running ASP.Net.

Please note, all data was captured with a 2-tap kegerator and a KegSmarts configured for such a system. A 3-tap system's settings are assumed below, but have not actually been captured or tested.

## HTTP Header Values

* Cache-Control: private
* Content-Length: (variable)
* Content-Type: text/html; charset=utf-8
* Server: Microsoft-IIS/10.0
* X-AspNetWebPages-Version: 3.0
* X-AspNet-Version: 4.0.30319
* Request-Context: appId=cid-v1:(unique-id)
* Access-Control-Expose-Headers: Request-Context
* X-Powered-By: ASP.NET
* Date: Day(Xxx), Date(dd) Month(Mon) Year(yyyy) hh:mm:ss GMT
* Connection: keep-alive

All responses from PicoBrew are preceded by this.

## Boot Up Sequence

KegSmarts goes through a series of steps with a restart. The first is the "notification" call.

##### Request

`GET /api/ks/notification?machine=<PID>&firm=<Version> HTTP/1.0
Host: www.picobrew.com
Connection: Keep-Alive`

Values:

* PID: The product ID for the KegSmarts, as reported PicoBrew's User->Settings->Equipment page.
* Version: The firmware version currently installed. (1.0.6 is the most recent)

Example:

```
GET /api/ks/notification?machine=1234ab123456&firm=1.0.6
```

##### Response

`##`

Next comes a "configure" call.

##### Request

`GET /api/ks/rev1/configure?machine=<PID>&sensors=<sensors>&mintemp=<mT>&tempData=<T> HTTP/1.0
Host: www.picobrew.com
Connection: Keep-Alive`

Values:

* PID: The product ID for the KegSmarts, as reported PicoBrew's User->Settings->Equipment page.
* sensors: strings in the format of "Location_UID_Type(_ID)" separated by a "!"
	* Location values appear to be:
		* 0 - Unassigned (KegSmarts Thermometer always has this value)
		* 1-4 - Taps, with an additional possible "keg" location for fermentation (a two-tap system has Tap 1, Tap 2, and Keg 3; a three-tap system has Tap 1, Tap 2, Tap 3, and Keg 4)
	* UID values are unique to each sensor:
		* ? - KegSmarts Thermometer (need more data from other units to look for patterns)
		* nnnxxc7190000 - KegPlates
		* ? - KegWarmer (need more data from other units to look for patterns)
	* Type values are a guess at this point:
		* 0 - KegSmarts Thermometer
		* 1 - KegPlate
		* 2 - KegWarmer
	* ID values appear for the user-added items, to match the GUI selection on the KegSmarts and the label on the PicoBrew web interface:
		* KegPlate - KegSmarts supports up to 4 (4 is reserved for a CO2 plate).
		* KegWarmer - KegSmarts supports up to 2, with the extended tail unit (1 with the default).
* mT: Temperature in Fahrenheit, appears to always be "32" (freezing).
* T: Temperature in Fahrenheit, the current user-selected temperature.
		
Example:

```
GET /api/ks/rev1/configure?machine=1234ab123456&sensors=0_bb60caa11603_0!3_123ac7190000_1_1!0_123bc7190000_1_2!3_aa1cbe931604_2_1&mintemp=32&tempData=65
```
This example on a 2-tap system has a thermometer, KegPlate 1 assigned to Keg 3, KegPlate 2 unassigned, and KegWarmer 1 assigned to Keg 3. 

##### Response

`##`

Next comes a "log" call.

##### Request

`GET /api/ks/rev1/log?machine=<PID>&data=<data> HTTP/1.0
Host: www.picobrew.com
Connection: Keep-Alive`

Values:

* PID: The product ID for the KegSmarts, as reported PicoBrew's User->Settings->Equipment page.
* data: strings in 3 format. The first is always the thermometer, while the others depend on if a keg is tapped or fermenting. All are separated by a "!".
	* Thermometer is in the format "0_T_0_0_0"
		* T - Current temperature in Fahrenheit of the kegerator.
	* Tapped is in the format "Location_kT_0_State_Weight"
		* Location - 1-4, for Taps and kegs (a two-tap system has Tap 1, Tap 2, and Keg 3; a three-tap system has Tap 1, Tap 2, Tap 3, and Keg 4).
		* kT - Current temperature in Fahrenheit of the keg/KegWarmer.
		* State - 1=tapped, unknown if others.
		* Weight - Adjusted weight of the beer only, in grams. The Fermentation call includes a value for the weight of the keg, which is subtracted from the raw number the KegPlate reports to the KegSmarts (as can be seen on the device via Home-> Configure KegSmarts-> Configure Equipment-> KegPlate-> Test KegPlate).
	* Fermenting is in the format "Location_kT_?_State_Weight_Fermentation_??_Time_fT"
		* Location - 1-4, for Taps and kegs (a two-tap system has Tap 1, Tap 2, and Keg 3; a three-tap system has Tap 1, Tap 2, Tap 3, and Keg 4).
		* kT - Current temperature in Fahrenheit of the keg/KegWarmer.
		* ? - A 3-digit value that appears to increase and decrease at times throughout the fermentation. Does not correspond to SG. Perhaps a more sensitive temperature reading to 3 decimals?
		* State - 2=fermenting, unknown if others.
		* Weight - Adjusted weight of the beer only, in grams. The Fermentation call includes a value for the weight of the keg, which is subtracted from the raw number the KegPlate reports to the KegSmarts (as can be seen on the device via Home-> Configure KegSmarts-> Configure Equipment-> KegPlate-> Test KegPlate).
		* Fermentation - Appears to be static.
		* ?? - Appears to be static at "1".
		* Time - Minutes remaining of fermentation session until the cold crash.
		* fT - Goal temperature in Fahrenheit of the fermentation session. 
		
Example 1:

```
GET /api/ks/rev1/log?machine=1234ab123456&data=0_50_0_0_0!1_50_0_1_17000!2_50_0_1_-3515! 
```
This example shows a kegerator temperature of 50F, Keg 1 is tapped with a temperature of 50F and an adjusted KegPlate reading of 17,000 grams, and Keg 2 is tapped with a temperature of 50F and an adjusted KegPlate reading -3,515 grams (as the KegPlate has nothing physically on it, and the keg's weight was set at 3,515 grams from the last Fermentation call).

Example 2:

```
GET /api/ks/rev1/log?machine=1234ab123456&data=0_64_0_0_0!3_72_609_2_18020_Fermentation_1_13692_70!  
```
This example shows a kegerator temperature of 64F, Keg 3 is fermenting with a temperature of 72F and an adjusted KegPlate reading of 18,020 grams, 13,692 minutes remaining until the cold crash, and a goal temperature of 70F.

##### Response

`#n#`

Values:

* 0 - Appears following routine periodic logging.
* 1 - Appears following a boot or a user-initiated change on the PicoBrew website, and seems to result in a ksget call.

With a boot, the response is always "#1#" and is always followed by a "ksget" call.

##### Request

`GET /api/ks/rev1/ksget?machine=<PID> HTTP/1.0
Host: www.picobrew.com
Connection: Keep-Alive`

Values:

* PID: The product ID for the KegSmarts, as reported PicoBrew's User->Settings->Equipment page.

##### Response

`    #NAME|TT|kT$(D)$(K)$#`

Values (NOTE: this is a fairly complicated string, so many of these are best guesses):

* NAME: The name of the KegSmarts.
* TT: Total physical taps present on the device.
* kT: Current goal temperature in Fahrenheit of the kegerator.
* $: Separator
* D: Device string in the format "Type|Location|ID"
	* Type - 4=KegPlate and 5=KegWarmer
	* Location - 1-4, for Taps and kegs (a two-tap system has Tap 1, Tap 2, and Keg 3; a three-tap system has Tap 1, Tap 2, Tap 3, and Keg 4).
	* ID - 1-4 for KegPlates, and 1-2 for KegWarmers (to match the GUI selection on the KegSmarts and the label on the PicoBrew web interface).
* K: Keg string in three formats. This data displays on the KegSmarts device. All start with "Type|Location"
	* Type - 0=Inactive, 1=Tapped, 2=Fermenting
	* Location - 1-4, for Taps and kegs (a two-tap system has Tap 1, Tap 2, and Keg 3; a three-tap system has Tap 1, Tap 2, Tap 3, and Keg 4).
	* Inactive - No further data included.
	* Tapped - Follows with "|ABV|IBU|Temp|user|Beer Name|Beer Style|Hops|Grains|Oz Remaining|DaysT|kWeight|Rating"
		* ABV - Alcohol by volume to 1 decimal. Pulled from the recipe.
		* IBU - International bitterness units. Pulled from the recipe.
		* Temp - Current serving temperature in Fahrenheit of keg/KegWarmer.
		* user - User who brewed the beer. Pulled from the recipe. 
		* Beer Name - Pulled from the recipe.
		* Beer Style - Pulled from the recipe.
		* Hops - Pulled from the recipe. 
		* Grains - Pulled from the recipe. Does not display on the device.
		* Oz Remaining - Converted from grams via adjusted KegPlate, or via "Dispense Menu-> Pour a Glass" selection on KegSmarts device.
		* DaysT - Time in days since the keg was tapped.
		* kWeight - Keg's weight in grams, to be subtracted from the raw KegPlate measurement recorded by the KegSmarts. Pulled from the recipe.
		* Rating = 0-5 out of 5. Selected via "Dispense Menu-> Ratings" selection on KegSmarts device or via the PicoBrew web interface.
	* Fermenting - Follows with "|Beer Name|hash|fT|?|Time|Ferment|Crash|Rating"
		* Beer Name - Pulled from the fermentation.
		* hash - Appears this is a unique ID generated from PicoBrew's server.
		* sT - Current set temperature in Fahrenheit of the fermentation session.
		* ? - Appears to be static at "1".
		* Time - Minutes remaining of fermentation session until the cold crash.
		* Ferment - Appears to be static "Fermentation, goal session temperature in Fahrenheit, total fermentation time in minutes". Pulled from the fermentation.
		* Crash - Appears to be static "Crash Chill, goal crash temperature in Fahrenheit, 0" 

Example 1:

```
    #KegSmart's|2|50$4|0|1$4|2|2$0|1$1|2|7.3|159|50|mstitzer|Stargazer Z Kit|American IPA|Cascade,Centennial,Columbus,Comet|American Two-Row Pale,CaraPils,Crystal 40L,Victory Malt|0|48|3515|0$0|3$# 
```
This example shows a kegerator named "KegSmart's" with 2 taps, a temperature of 50F, KegPlate 1 is unassigned, KegPlate 2 is assigned to Tap 2, Tap 1 is inactive; Tap 2 has Stargazer Z Kit tapped, with 0 oz. remaining, 48 days on tap, a keg weight of 3515 grams, and a 0/5 rating; and Keg 3 is inactive.

Example 2:

```
    #KegSmart's|2|66$4|2|2$4|3|1$5|3|1$0|1$2|2|NB Honey Kolsch|8923588A4D614A8791E997AF931CE237|66|1|13535|Fermentation,66,14400|Crash Chill,44,0$2|3|Black is Beautiful|CC237F3C110847FDA1B01667971566B7|71|1|12368|Fermentation,70,14400|Crash Chill,44,0$# 
```
This example shows a kegerator named "KegSmart's" with 2 taps, a temperature of 66F, KegPlate 2 is assigned to Keg 2, KegPlate 1 is assigned to Keg 3, KegWarmer 1 is assigned to Keg 3, Tap 1 is inactive; Keg 2 has NB Honey Kolsch fermenting, with a current temperature of 66F set, 13,535 minutes of the fermentation session remaining, the fermentation has a goal temperature of 66F and a duration of 14,400 minutes, and a cold crash goal temperature of 44F; and Keg 3 has Black is Beautiful fermenting, with a current temperature of 71F set, 12,368 minutes of the fermentation session remaining, the fermentation has a goal temperature of 70F and a duration of 14,400 minutes, and a cold crash goal temperature of 44F.

It seems that every "ksget" call is followed by a "ksacknowledge" call, including at boot.

##### Request

`GET /api/ks/rev1/ksacknowledge?machine=<PID> HTTP/1.0
Host: www.picobrew.com
Connection: Keep-Alive`

Values:

* PID: The product ID for the KegSmarts, as reported PicoBrew's User->Settings->Equipment page.

##### Response

`#n#`

Values:

* 0 - Only value seen so far.

## configure

In addition to being called at boot, appears to be called when the user adds, removes, or assigns hardware via the KegSmarts device. All known details described above during boot.

## ferment

A two-step process. The first step is initiated via "Main Menu-> Start Fermenting" on the KegSmarts device. The second step is initiated by selecting an option from the list that appears.

##### Request - Step 1

`GET /api/ks/rev1/ferment?machine=<PID>code=<code> HTTP/1.0
Host: www.picobrew.com
Connection: Keep-Alive`

Values:

* PID: The product ID for the KegSmarts, as reported PicoBrew's User->Settings->Equipment page.
* code: 0=pull list from server 1=pull session info

##### Response - Step 1

`#Beer Name1/hash1|Beer Name2/hash1|#`

Values:

* Beer Name: "Name" if brewed but not fermented, "*Name" if brewed and fermented.
* hash1: Appears this is a unique ID generated from PicoBrew's server. Also not the same hash as used in ksget's fermenting keg string.

Example 1:

```
#NB Honey Kolsch/AAAD6571BAA847CAACBFF24DA7AFED8A|Black is Beautiful/1E2FCAA36AB84A56B8F6C1F84C14602F|*Stargazer Z Kit/EC0B600CDA884D608A187DBB74608D84|*Black Mamba/3BA60A02EBF0477791443563CCDA38F3|*Brickwarmer Red/EFB0D5D9091647428D9D183FBD2B281E#
```
The Beer Names then display as a list of possible fermentation sessions to select from on the KegSmarts device.

##### Request - Step 2
`GET /api/ks/rev1/ferment?machine=<PID>&code=<code>&session=<hash1>&tap=<Location> HTTP/1.0
Host: www.picobrew.com
Connection: Keep-Alive`

Values:

* PID: The product ID for the KegSmarts, as reported PicoBrew's User->Settings->Equipment page.
* code: 0=pull list from server 1=pull session info
* hash1: The unique ID from the list in step 1.
* Location - 1-4, for Taps and kegs (a two-tap system has Tap 1, Tap 2, and Keg 3; a three-tap system has Tap 1, Tap 2, Tap 3, and Keg 4).

Example 1:

```
GET /api/ks/rev1/ferment?machine=1234ab123456&code=1&session=1E2FCAA36AB84A56B8F6C1F84C14602F&tap=3
```

This is the result of selecting to start fermenting "Black is Beautiful" from the list which displays after step 1.

##### Response - Step 2

`#Fermentation/fT/Time|Crash Chill/cT/0#`

Values:
* Fermentation: Appears to be static.
* fT: Goal temperature in Fahrenheit of the fermentation session. 
* Time: Minutes remaining of fermentation session until the cold crash.
* Crash Chill: Appears to be static.
* cT: Goal temperature in Farenheit of the cold crash.
* 0: Appears to be static.

Example 1:

```
#Fermentation/70/14400|Crash Chill/44/0#
```
These values are for "Black is Beautiful".

## ksacknowledge

As above, only seems to be called after ksget. All known details described above during boot.

## ksget

In addition to being called every boot, appears to be called when changes on the server prompt a response to "log" calls with "#1#". All known details described above during boot.

## log

In addition to being called every boot, KegSmarts calls this every 30-60 seconds to log all active sensor data related to tapped and fermenting kegs. Data is used on both the KegSmarts device and the PicoBrew server, including for fermentation graphs. All known details described above during boot.

## notification

As above, only seems to be called on boot. All known details described above during boot.
