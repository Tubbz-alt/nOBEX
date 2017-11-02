# nOBEX

## Introduction
nOBEX allows emulating the PBAP, MAP, and HFP profiles to test vehicle infotainment systems and
similar devices using these profiles. nOBEX provides PBAP and MAP clients to clone the genuine
virtual filesystems for these profiles from a real phones. This means downloading the entire phone
book and all text messages. Raw vcards, XML listings, and MAP BMSG structures are stored, and can
be modified as desired for negative testing. nOBEX can then act as a PBAP and MAP server, allowing
vehicles and other devices to connect to it and retrieve phone book and message information. Vcards,
BMSGs, and XML listings are sent exactly as saved, allowing malformed user modified data to go
through. Since most vehicle head units require HFP support before they attempt using PBAP and MAP,
nOBEX also provides rudimentary support for HFP. It will send back user customizable preset
responses to AT commands coming from the vehicle's head unit. This allows mimicking a real cell
phone.

nOBEX is built on top of the [PyOBEX](https://bitbucket.org/dboddie/pyobex) project by David Boddie.
This tool would not have been possible without David's great efforts in making OBEX approachable
and easy to work with. nOBEX extends PyOBEX by adding support for large multi-part OBEX messages,
HFP emulation, PBAP and MAP servers, a MAP client, and an improved PBAP client.

nOBEX (and PyOBEX) use the BlueZ Bluetooth stack to advertise over the Service Discovery Protocol
(SDP) and establish RFCOMM connections. nOBEX/PyOBEX contain standalone implementations of the
OBEX specification for client and server roles. [PyBluez](https://github.com/karulis/pybluez) is
used to access the BlueZ API from Python. Both Python 2 and 3 are supported.

In client mode, nOBEX uses BlueZ to query services offered by the server. If it detects that the
requested service is available, it connect to the server over RFCOMM on the port specified over
SDP. OBEX requests are constructed and sent to the server in accordance with the profile in use.
Responses are interpreted and saved to disk. Client modes for PBAP and MAP can be used to clone a
real phone.

In server mode, nOBEX advertises the available services over SDP. When a client makes an RFCOMM
connection on the advertised port, the server will accept and handle OBEX requests. OBEX responses
to requests will be sent using the data on disk. The PBAP and MAP servers serve file/folder
structures matching those generated by the respective clients.

## Installation Instructions
The following setup instructions were tested on Fedora 24. Other recent distributions may also
work, but experiences may vary. Recent changes to BlueZ may have broken nOBEX, and Fedora 26
is known to be problematic for running OBEX servers with nOBEX. If all else fails, use
[Fedora 24](https://archive.fedoraproject.org/pub/archive/fedora/linux/releases/24/Workstation/x86_64/iso/Fedora-Workstation-Live-x86_64-24-1.2.iso).
Also be aware that OBEX servers tend to not work properly in virtual machines, so please use a
native install of Linux.

First install pybluez:
```
sudo dnf install python3-bluez
```

Try probing advertised local services on SDP:
```
sudo sdptool browse local
```

If you are running a recent distribution (like Fedora 24), it will probably fail due to some
breaking API changes in new versions of bluez. You can fix it by running bluetoothd in compat mode.
Do this by editing the systemd service for bluetoothd.
```
sudo vi /usr/lib/systemd/system/bluetooth.service
```

Add --compat to the ExecStart line:
```
ExecStart=/usr/libexec/bluetooth/bluetoothd --compat
```

Now restart bluetoothd:
```
sudo service bluetooth stop
sudo systemctl daemon-reload
sudo service bluetooth start
sudo hciconfig -a hci0 reset
```

Test browsing local SDP services again (it should work this time):
```
sudo sdptool browse local
```

Get nOBEX and install it:
```
 git clone https://github.com/nccgroup/nOBEX.git
 cd nOBEX
 sudo python3 setup.py install
```

## Usage Instructions
### PBAP
Find the MAC address of a phone whose phone book you wish to clone:
```
hcitool scan
```

Clone the PBAP contents of an existing phone (use your correct MAC and a preferably empty or
nonexistent destination directory of your choice):
```
python3 examples/pbapclient-download.py 5C:51:88:8A:EC:5B ~/pbap_root/
```

Modify the vcards and listing XMLs in the your dump destination directory as desired. Now run a
PBAP server using the cloned phone book:
```
sudo python3 examples/multiserver.py --pbap ~/pbap_root/
```

You'll also need to pair your PBAP client with the computer (PBAP server).

### MAP
Pull the message data off your phone to establish a test MAP tree:
```
python3 examples/mapclient-download.py 5C:51:88:8A:EC:5B ~/map_root/
```

Alternatively, if your phone doesn't support MAP properly, use the MAP sample data tree
located in the `examples/map_root` folder.

Modify the sample data as desired. Then run the server, indicating where it should look for the
root of the MAP tree.
```
 sudo python3 examples/multiserver.py --map ~/map_root/
```

### HFP
The HFP server (audio gateway) is fairly basic, sending back preconfigured replies to select
commands. The server is set up to support common HFP commands out of the box, but every vehicle
will likely require a few additional commands and/or changes to responses. Custom responses can be
configured through a format of a command and response pairs on each line, command and response
being separated by a tab. Sample config files can be found in the examples/bbeast folder.

To run a standalone HFP server (config file is optional):
```
sudo python3 examples/multiserver.py --hfp [config_file]
```

### Combining servers
The multiserver.py scripts allows running any combination of HFP, MAP, and PBAP servers
simultaneously. Just combine the arguments from the examples shown above. To run HFP, MAP, and
PBAP simultaneously:
```
python3 examples/multiserver.py --map ~/maproot/ --pbap ~/pbap_root/ --hfp [config_file]
```

The combination of HFP and PBAP has been tested successfully on a 2012 Ford Focus.

## Applications
The primary purpose of nOBEX is to perform negative testing and fuzzing of PBAP and MAP clients on
automotive head units. The HFP support and PBAP/MAP client support are intended to facilitate this
goal. Manual fuzzing can be performed by running a server with hand-modified XML listings, vcards,
and BMSGs. OBEX is a rich fuzzing target with many nested TLV structures that can span multi-part
messages. PBAP and MAP greatly increase the attack surface with vcard, BMSG, and XML parsers.

nOBEX does not have integrated support for automated fuzzing, but since it is written in Python,
it is easy to extend. More powerful fuzzing capabilities can be built by pairing it with a mutation
engine and instrumenting the target device.

Beyond fuzzing MAP and PBAP on automotive head units, nOBEX can also be used for normal positive
testing of PBAP, MAP, and other OBEX profiles (such as FTP) for both client and server roles. The
PBAP and MAP servers were tested with the [http://intradarma.com/ OBEX Commander] app for Android,
in which many crashes could be triggered by faulty OBEX communication and malformed profile specific
data. Furthermore, the HFP support can be used to manually fuzz AT commands.
