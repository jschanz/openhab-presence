# openhab-presence

(working) presence detection for openHAB

This example describes a presence detection solution for openHAB. I've found a lot of other ideas and suggestion in the Internet, but none of them was working for me and my environment. So here is solution thousand and X ...
The presence solution here is based on the networkhealth-binding. No magic dust like some awesome Fritz!Box hacks are needed. Everything ist based on plain ICMP (if openHAB running as root), otherwise Java would do a tcp connect to port 7 (echo port). This could sometimes create little bit strange behaviors.  

Long story short. You need first an presence.item file in your item configuration folder:

In this example two mobile devices (Dave and Alice) are created.

```
Group gMobiles

Switch Presence				"Status" 				<users>

/* Dave */
Switch PresenceDaveMobile	  "Handy Dave"		<ethernet>					{nh="192.168.178.100"}
Switch PresenceDave 		    "Dave"					<man> 		(gMobiles)

/* Alice */
Switch PresenceAliceMobile	  "Handy Alice"		<ethernet>					{nh="192.168.178.100"}
Switch PresenceAlice 		    "Alice"					<woman> 	(gMobiles)

```

Change the ips from above to your ips.

Next you can create a presence.sitemap or include the code from presence.sitemap in your sitemap.

```
sitemap Smarthome label="Hauptmenue" {
  Text label="Anwesenheit" icon="users" {
    Frame label="Anwesentheitsstatus" {
      Switch item=Presence label="Status"
      Switch item=PresenceDave label="Dave"
      Switch item=PresenceAlice label="Alice"
    }

    Frame label="Mobiltelefone" {
      Switch item=PresenceDaveMobile
      Switch item=PresenceAliceMobile
    }
  }
}
```

If everything went fine during the configuration reload, you can test the mobile phones. Disable the wifi on Dave's mobile phone an have a look on the switch PresenceDaveMobile under "Mobiltelefone". Does it change to OFF? So everything is fine. Otherwise you may have a deeper look in openhab.log and events.log what happend in openHAB.

Next we create the rule set for setting the presence states of the person. We use here a "doubled" switch, because mobile phones sometimes loose their network connectivity and nothing is more stupid than some unwanted bling-bling-effects from your smarthome. To avoid this, the rule sets the presence to state off 5 minutes after the mobile phone has lost the network connectivity. You could try shorter values here (2 minutes are also working in our smarthome), but with 5 minutes, you are on the safe side.

So what happens in the rule set?

```
import org.openhab.core.library.types.*
import org.openhab.core.persistence.*
import org.openhab.model.script.actions.*
import java.lang.Math
``` 

First, we import the usual stuff  ... Then, we have to create a timer for every mobile phone in the network. This timer helps to avoid flapping.

```
// Timer um kurzzeitige Verbindungabbrüche zu puffern
var Timer t_presence_Dave_absent
var Timer t_presence_Alice_absent
```

This rule sets the global presence state to detect if anyone is at home. Some magic here, because i've got this from the openHAB documentation. It works very well :-)
Actually a pushover-message is sent to my mobile phone if the presence state changes.

``` 
rule "Anwesenheit jede Minute prüfen"
when
    Time cron "0 */1 * * * ?"
then
  if (Presence.state == ON) {
    logInfo("smarthome.rules", "check ... "+ gMobiles.members.toString)

    if(gMobiles.members.filter(s | s.state == ON).size == 0) {
      logInfo("smarthome.rules", "Abwesenheit erkannt. Keiner Zuhause!")
      pushover("Abwesenheit erkannt. Keiner Zuhause!")
      sendCommand(Presence, OFF)
    }
  } else {
    //For initialisation. If Presence is undefined or off, although it should be on.
    if(gMobiles.members.filter(s | s.state == ON).size > 0) {
      sendCommand(Presence, ON)
    } else if (Presence.state == Undefined || Presence.state == Uninitialized) {
      sendCommand(Presence, OFF)
    }
  }
end
```

This rule is sets also the global presence state to ON if a mobile phone is found. Curiously I need this rule, because the one from above doesn't catch all states.

```
rule "Anwesenheit setzen, wenn mindestens ein Mobiltelefon gefunden wurde"
when
  Item gMobiles changed
then
  if (Presence.state != ON) {
    if(gMobiles.members.filter(s | s.state == ON).size > 0) {
      logInfo("presence.rules", "Ein oder mehr Mobiletelefone im WLAN gefunden. Anwesenheit setzen.")
      sendCommand(Presence, ON)
    }
  }
end
```

Let's move on with Dave. The comments in the rule should describe the workflow. On every update the rule is triggerd and checks for state OFF and ON. The examples from the openHAB documentation didn't work for me. There the when clause is done by "changed from OFF to ON" and "change from ON to OFF".
If the mobile phone receives an update to OFF, a trigger (if not already running) is created. If the mobile phone recives an update to ON, a trigger (if running) is canceled.

```
rule "Statusupdate PresenceDaveMobile"
when
  Item PresenceDaveMobile received update
then
	if (PresenceDaveMobile.state == OFF) {
    logInfo("presence.rules", "Statusupdate: PresenceDaveMobile ist gleich OFF")

    // Abwesenheitstimer starten, wenn nicht bereits einer gesetzt wurde und PresenceDave ON ist
		if (PresenceDave.state == ON) {
      logInfo("presence.rules", "PresenceDave (ON) && PresenceDaveMobile(OFF): Timer-Status -> " + t_presence_Dave_absent)

      if ( (t_presence_Dave_absent == null) ) {
        logInfo("presence.rules", "Dave S5 nicht im WLAN gefunden -> Kein Timer gesetzt. Abwesenheitstimer t_presence_Dave_absent setzen")
				t_presence_Dave_absent = createTimer(now.plusMinutes(5)) [|
					logInfo("presence.rules", "Abwesenheitstimer t_presence_Dave_absent abgelaufen -> PresenceDave von ON auf OFF")
					sendCommand(PresenceDave, OFF)
				]
			}
		}

	} else if (PresenceDaveMobile.state == ON) {

		logInfo("presence.rules", "Statusupdate: PresenceDaveMobile ist gleich ON")

		// Abwesenheitstimer prüfen und ggf. abbrechen
		logInfo("presence.rules", "PresenceDaveMobile(ON): Timer-Status -> " + t_presence_Dave_absent)
		if (t_presence_Dave_absent != null) {
			logInfo("presence.rules", "Abwesenheitstimer t_presence_Dave_absent abbrechen. (Flapping)")
			t_presence_Dave_absent.cancel
			t_presence_Dave_absent = null
		} else {
			logInfo("presence.rules", "Abwesenheitstimer t_presence_Dave_absent ist nicht aktiv")
		}

		// Anwesenheitsstatus auf ON setzen, falls im Status OFF
		if (PresenceDave.state == OFF) {
			logInfo("presence.rules", "Dave S5 im WLAN gefunden -> PresenceDave: OFF -> ON")
			sendCommand(PresenceDave, ON)
		}

	} else {
		logError("presence.rules", "Statusupdate PresenceDaveMobile UNDEFINED")
	}
end
``` 

Both following rules are not necessary, but very helpful if you would create some events if Dave leaves or returns.

```
rule "Statusupdate PresenceDave -> OFF"
	when
		Item PresenceDave received command OFF
	then
		logInfo("presence.rules", "Dave hat das Haus verlassen ... (PresenceDave -> OFF)")
		pushover("Dave hat das Haus verlassen ... (PresenceDave -> OFF)")
	end
```

```
rule "Statusupdate PresenceDave -> ON"
	when
		Item PresenceDave received command ON
	then
		logInfo("presence.rules", "Dave kommt nach Hause ... (PresenceDave -> ON)")
		pushover("Dave kommt nach Hause ... (PresenceDave -> ON)")
	end
```
