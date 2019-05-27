# alsa_loopback_mpd_and_icecast2
Setting up an ALSA loopback, pipelining it to mpd.service, and then relaying it to icecast2.service (Just a howto for now)


# Setting up an ALSA loopback, pipelining it to mpd.service, and then relaying it to icecast2.service

## 1. How to determine hw:X1,Y1

First, check for soundcards available in your system in /proc/asound/cards. 
Below an example:
```
# cat /proc/asound/cards

 0 [ALSA           ]: bcm2835 - bcm2835 ALSA
                      bcm2835 ALSA
```

In the above case, the principal card is "0 [PCH]: HDA-Intel - HDA Intel PCH" for which the index X1 is 0. 
Usually the needed alsa soundcard (hw:X1,X2) is found at (hw:0,0)

## 2. Creating a custom ~/.asoundrc file
the file ```~/.asoundrc``` should be filled as shown below. 
Change the fields "X1" and "X2" at pcm.loopin { ... } according to the results of step one. 

> Note that this file may not exist and it has to be owned by the process owner of the sound source (eg. yt, musicplayer)

```
# ~/.asoundrc
pcm.multi {
    type route;
    slave.pcm {
        type multi;
        slaves.a.pcm "output";
        slaves.b.pcm "loopin";
        slaves.a.channels 2;
        slaves.b.channels 2;
        bindings.0.slave a;
        bindings.0.channel 0;
        bindings.1.slave a;
        bindings.1.channel 1;
        bindings.2.slave b;
        bindings.2.channel 0;
        bindings.3.slave b;
        bindings.3.channel 1;
    }

    ttable.0.0 1;
    ttable.1.1 1;
    ttable.0.2 1;
    ttable.1.3 1;
}

pcm.!default {
    type plug
    slave.pcm "multi"
}

ctl.!default {
    type hw
    card ALSA
} 

pcm.output {
    type hw
    card ALSA
}

pcm.loopin {
    type plug
    slave.pcm "hw:Loopback,X1,X2"
}

pcm.loopout {
    type plug
    slave.pcm "hw:Loopback,1,0"
}
```
[ref: ~/.asoundrc](https://raspberrypi.stackexchange.com/a/57801)

## 3. Modifying the /etc/mpd.conf file

To connect the icecast2.service with the mpd.service, add the following parts to your ```/etc/mpd.conf``` and customize settings as you want them.

```
# /etc/mpd.conf

# FOR THE HTTPD SERVER 
# (NOTE THAT THERE IS NO AUTH CONTROL, SO INTRANET ONLY IS ADVISABLE)
# THIS SERVER IS GOOD TO TEST IF THE LOOPBACK BRIDGE IS WORKING
# AFTER STARTING THE MPD.SERVICE YOU SHOULD BE ABLE TO CONNECT WITH A CLIENT (EG. VLC, browser)
audio_output {
type                "httpd"
name                "My HTTP Stream"
encoder             "vorbis"            # optional, vorbis or lame
port                "8000"
#bind_to_address    "192.168.1.42"      # optional, IPv4 or IPv6
#quality            "5.0"               # do not define if bitrate is defined
bitrate             "128"               # do not define if quality is defined
format              "44100:16:1"
#max_clients        "1"                 # optional 0=no limit
always_on           "yes"
}

# FOR ICECAST SERVER WITH PASSWORD PROTECTION
audio_output {
        type            "shout"
        encoding        "ogg"                   # optional
        name            "My Shout Stream"
        host            "localhost"
        port            "8000"
        mount           "/mpd.ogg"
        password        "VERY PRIVATE PASSWORD FROM ICECAST XML FILE (<source-password>)"
#       quality         "5.0"
        bitrate         "128" #"92" #"64"
        format          "44100:16:1"
        protocol        "icecast2"              # optional
        user            "source"                # optional
        always_on       "yes"
#       description     "My Stream Description" # optional
#       url             "http://example.com"    # optional
#       genre           "jazz"                  # optional
#       public          "no"                    # optional
#       timeout         "2"                     # optional
#       mixer_type      "software"              # optional
}

```

## 4. How to determine ```hw:X2,Y2``` and the Loopback device

We now need to load the snd-aloop module:
```sudo modprobe snd-aloop```
This will create a new virtual device called Loopback that we above referenced as ```hw:X2,Y2```:

```
# cat /proc/asound/cards

 0 [ALSA           ]: bcm2835 - bcm2835 ALSA
                      bcm2835 ALSA
 1 [Loopback       ]: Loopback - Loopback
                      Loopback 1
```

As it can be seen, Loopback has an index X2 equals 1. 
This device is then ```hw:1,1```

>Note that this change is not permanent.
>If you want to make it permanent, you have to add ```snd-aloop``` to:
>```/etc/modules```

To add the virtual loopback device as item to the mpd playlist, run the following command:
```mpc add alsa://hw:1,1```

## 5. Setting up the icecast2 service

Just install icecast2 and follow the setup (settings such as passwords,ports, etc can be customized later)

To enable htacess protection, you can add these lines to the file ```/etc/icecast2/icecast.xml```: 

```
<!--/etc/icecast2/icecast.xml-->
<mount>
        <mount-name>/mpd.ogg</mount-name>
        <authentication type="htpasswd">
            <option name="filename" value="ABSOLUTE PATH TO AUTHFILE"/>
            option name="allow_duplicate_users" value="0"/>
        </authentication>
</mount>
```

After doing this modification, you will also have to do the following steps
1. create the ```AUTHFILE``` somewhere
```touch /etc/icecast2/AUTHFILE```
2. chown the ```AUTHFILE``` to the user that is running the icecast2 server
```chown icecast2 /etc/icecast2/AUTHFILE```
3. (re-/start) the icecast2 server (YOU DON'T NEED TO DO THIS STEP AT THIS POINT, PLEASE READ ON)

From now on, you should be able to edit user accounts over the administration framework of your icecast2 server with a browser.

## 6. Finishing it up

To start everything altogether run the following command:
```sudo systemctl restart icecast2.service mpd.service```

To check if everything is working accordingly, run the following command:
```sudo systemctl status icecast2.service mpd.service -l```

To start playback of mpd run the following command:
```mpc play # this will start playback of the first track in the playlist```

> you could also use graphical tools to control mpd.service behaviour, such as gmpc.

You should be able to listen to any sound from your server locally and remotely at the same time from now on.
In order to test it, start playback of sounds in any program you like, such as chromium. 

Word of advice:
- Some programs (eg. chromium) might get confused and choose the virtual loopback device as soundcard, hence blocking the mpd.service from accessing the virtual loopback device (you won't be able to hear anything in that case)
- ```The mpd.service should in any case be running and hooked to the virtual loopback device, before any sound producing programs have been started.```
