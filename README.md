eti-tools
============

This is a software collection for converting [Ensemble Transport Interface](http://wiki.opendigitalradio.org/Ensemble_Transport_Interface) used in terrestrial Digital Audio Broadcasting (DAB/DAB+/T-DMB).

The main purpose of these apps is to convert/manipulate ETI-NA/ETI-NI streams (by using pipelines). With these software tools you can create your own IceCast2 internet-radio server which will use your local DAB/DAB+ transmitter as source for the stations streams. You also can re-multiplex some (needed) stations from one ETI-stream to another by using ZeroMQ feature of the ni2http application and [ODR-DabMUX](https://github.com/Opendigitalradio/ODR-DabMux). 

This software also allows to receive and convert special formatted [Satellite DAB(+) streams](#satellite-dab-feeds) (so-called feeds) into regular ETI-NI which then can be used to play in [dablin](https://github.com/Opendigitalradio/dablin) or even feed modulator software/hardware (check local laws!) or to create internet-station from that source.


Table of Content
----------------

* [Prerequisitions](#prerequisitions)
* [Installation](#installation)
* Tools
  * [ts2na](#ts2na)
  * [na2ni](#eti-na2ni)
  * [edi2eti](#eti-edi2eti)
  * [eti2zmq](#eti-eti2zmq)
  * [ni2http](#eti-ni2http)
* [Satellite DAB(+) feeds](#satellite-dab-feeds)
  * [Guide](#guide)
    * [dvbstream](#dvbstream)
    * [Sat>IP](#alternative)


Prerequisitions
----------------

Additional libraries which are needed:

* cmake (needed for libfec)

  For Debian (incl. Ubuntu and derivates)

      sudo apt-get install cmake


* [libfec](https://github.com/Opendigitalradio/ka9q-fec) - for Reed-Solomon FEC, may be enabled/disabled.

      git clone https://github.com/Opendigitalradio/ka9q-fec.git
      cd ka9q-fec/
      mkdir build
      cd build/
      cmake ..
      make
      sudo make install
      
   To clean the build directory type
       
      rm -rf *

* [libshout](http://www.icecast.org/download.php) - for NI-to-HTTP converter (included in this package library is modified to support aac and raw streaming).
  
  For Debian (incl. Ubuntu and derivates)

      sudo apt-get install icecast2

  and follow the instructions

* [libzmq](http://zeromq.org) - optional: for ZeroMQ output of NI-to-HTTP converter (possible re-mux of ETI-streams containing DAB/DAB+ streams). Comment it in `Makefile` if this option is not needed (see below).

Installation
----------------
    
    git clone https://github.com/piratfm/eti-tools.git
    cd eti-tools/

If you need to disable ZeroMQ (see above): Edit `Makefile` and comment (= add `#` in front of) lines 17 and 18, then

    make

As there is no `make install` you find the executables in the current folder. You could add aliases in `~/.bash_aliases` or add the eti-tools-directory to your `$PATH`.


ts2na
----------------

**ts2na** is a MPEG-TS to ETI-NA converter for satellite DAB(+) feeds.

**ts2na_dreambox.c** is a special version for Dreambox DM-500S which can be used to tune the frontend to a specific frequency. On a regular PC use [dvbstream](https://www.linuxtv.org/wiki/index.php/Dvbstream) or [MuMuDVB](https://www.linuxtv.org/wiki/index.php/Mumudvb) application to dump to `ts2na`.

    usage: ./ts2na [-p pid] [-s offset] [-i <inputfile>] [-o <outputfile>]

Default for `offset` is 12 bytes. If you get 

    ERROR: Can't find sync

try one of these (currently in Europe used) values for `offset`: 0, 12 or -3. 

Default for `pid` is 1062. Values for `pid` can be any other PID carrying an ETI-NA stream (e.g. 1061). In case of negative offset (`-s -3`, see above) this `pid` argument will be ignored as the DVB-S stream itself is no valid transport stream.

The output stream will be raw PID content = ETI-NA (G.704). The parameter [-s offset] must be seen in a MPEG-TS dump, in most cases that is unused 0xFF at the beginning of the each TS-packet's payload.


ETI na2ni
----------------

**na2ni** is an ETI-NA (G.704) to ETI-NI (G.703) converter. This tool automatically detects E1-sync bits in bitstream and inversion flag. Also it extracts ETI-LI content of the stream and encapsulate it into ETI-NI frames. It is possible to disable Reed-Solomon error correction, then the conversion speed will be dramatically increased.

    usage: ./na2ni [--no-fec] [-i <inputfile>] [-o <outputfile>]

The output stream will be 6144-bytes aligned raw ETI-NI stream (G.703)


ETI edi2eti
----------------

**edi2eti** is an EDI-AF or EDI-PF to ETI-NI converter. This tool automatically detects the AF/PF packet type. Maximal deinterleaving depth is set to 192ms in order to prevent high memory usage. It also extracts an ETI-LI content of the stream and encapsulate it into ETI-NI frames.

The tool is able to receive multicast data and save the converted stream into an ETI-file or publish it by ZeroMQ protcol. It is written to convert microwave links (WiFi or raw packet stream) to tcp-zeromq stream useable by [EasyDABv2 module](http://tipok.org.ua/node/46). It can also be used for satellite feeds on Eutelsat 7° East.

    usage: ./edi2eti [-o <outputfile|zeromq-uri>] [ip:port]

The output stream will be 6144-bytes aligned raw ETI-NI stream or local-port published ZeroMQ packet.

Sample of receiving multicast stream and convert it to ZeroMQ:

    ./edi2eti -o "zmq+tcp://*:18982"  232.20.10.1:12000

Sample of receiving multicast stream and save it to file:

    ./edi2eti -o "out.eti"  232.20.10.1:12000

ETI eti2zmq
----------------

**eti2zmq** is an ETI-NI to ZeroMQ converter. This tool plays a ETI-file and publishes it as server, just like [ODR-DabMux](https://github.com/Opendigitalradio/ODR-DabMux). It simulates pseudo-realtime streaming by adding a proper delay between sent frames. It also allows to play files in a loop.

    usage: ./eti2zmq [-i <input-file.eti>]-o <zeromq-uri>

The input stream must be 6144-bytes aligned raw ETI-NI.

Sample of playing "kbs.eti" file in-a-loop with pseudo-realtime streaming and app's activity indication:

    ./eti2zmq -i kbs.eti -a -l -d -o "zmq+tcp://*:18982"


ETI ni2http
----------------

**ni2http** is an ETI-NI to HTTP or ZeroMQ converter. This tool converts an eti-stream to mp2 resp. AAC and relays it to icecast2 or ODR-DabMUX server.

ETI-NI streams from terrestrial DAB(+) ensembles can also be created by [eti-stuff](https://github.com/JvanKatwijk/eti-stuff) or [dabtools](https://github.com/Opendigitalradio/dabtools). For satellite feeds see above.

    usage: ./ni2http [--list] [--delay] [-i <inputfile>] [-c <config_file>] [-s <SID>]

Use `--list` option to find SIDs and station names inside the ETI stream. If you wish to write the stream to stdout, then use `ni2http --sid <SID>`. In this case the config_file isn't needed.
The `--delay` option has to be used for offline-relaying (from the file, not from the stream). So in that case the application will wait 24ms after each ETI frame in order to make pseudo-realtime streaming.

The application is also able to parse FIC for auto-detecting of station name and X-PAD of DAB and DAB+ for setting current DLS (song titles).

Config sample:

    [server]
    host:       localhost
    port:       8000
    user:       source
    password:   hackme
    
    [channel]
    mount:      r5_live
    sid:        0xc228
    
    [channel]
    name:       Custom channel name2
    mount:      r1
    sid:        0xc221
    
    [channel]
    #stream name will be auto-detected
    #name:       Custom channel name3
    mount:      r6music
    sid:        0xc22b
    # extract_pad - use DLS info as icecast metadata, enabled by default
    extract_pad: 1
    #extract_dabplus - converts DAB+ stream into AAC-ADTS,
    #which is playable by the internet-radio players, enabled by default
    #If this option is disabled, this stream can be directly passed to ODR-DabMod
    extract_dabplus: 1
    
    [channel]
    # just write to file, no streaming to server.
    sid:        0xc223
    extract_pad: 0
    file:       /run/station7.fifo

    [channel]
    # pass DAB/DAB+ stream to ODR-DabMUX.
    sid:        0xc224
    extract_pad: 0
    zmq:       tcp://127.0.0.1:9001

In `[server]` section the parameters of Icecast2 server must be set.
In `[channel]` sections at least service ID of the channel must be presented.

If you wish to write the stream to a file, then use `file` to specify its location. If streaming to Icecast2 server is needed, then specify mount-point on the icecast server.

If you wish to re-stream to ODR-DabMUX then set the destination of the ZeroMQ URI to muxing server. To get a list of audio service IDs, use `ni2http --list -i <inputfile>`


Satellite DAB(+) feeds
----------------

This is an exclusive list of satellite feeds that you can use with `ts2na`, `edi2eti` (both from these `eti-tools`) or with [tsniv2ni](https://github.com/newspaperman/tsniv2ni) which works for ETI-NA(V.11).

Please note that you need `eti-tools` from June 2018 or later for EDI.

### DAB-Ensembles working with eti-tools

The format is MPEG-TS, which you have to convert into ETI-NA and then to ETI-NI.

Ensemble | Country | Sat | Freq | SR/FEC | Modulation | PID | SID | Offset
-- | -- | -- | -- | -- | -- | -- | -- | -- 
Bundesmux 5C | Germany | 23.5ºE | 12641V | 1342 5/6 | QPSK/DVB-S | 8192 | -- | -3
WDR 11D | Germany | 23.5ºE | 12645V | 1489 3/4 | QPSK/DVB-S | 8192 | -- | -3
BBC DAB | UK | 4.5ºE | 12303H | 25546 7/8 | QPSK/DVB-S | 1061 | 70 | 12
D1 DAB | UK | 4.5ºE | 12303H | 25546 7/8 | QPSK/DVB-S | 1062 | 60 | 12
SDL NATL | UK  | 4.5ºE | 12303H | 25546 7/8 | QPSK/DVB-S | 1063 | 80 | 12
D1 DAB | UK | 28.2ºE | 11425H | 27500 2/3 | QPSK/DVB-S | 1062 | 10585 | 12
SDL NATL | UK  | 28.2ºE | 11425H | 27500 2/3 | QPSK/DVB-S | 1063 | 10590 | 12
RAI DAB+ | Italy | 5.0°W | 11013V | 30000 3/5 | 8PSK/DVB-S2 ACM Multistream 11 PLS: Root/16416 or PLS: Gold/131070 | 1000 | -- | 0
TRT DAB+ | Turkey | 42.0E | 10953V | 1800 3/4 | QPSK/DVB-S | 1068 | -- | 0 

Notes: TRT DAB+ is already an NI stream! There is no need to use `na2ni` in this case! For Bundesmux and WDR you need to stream/save the complete transponder (PID 8192) as they don't have a transport stream. The RAI DAB+ only can received with a receiver/DVB card supporting ACM Multistream.
  

### DAB-Ensembles working with eti-tools

The format is EDI. 

All of them are from Germany and can be found on 7.0ºE, 12572V, Symbol rate 10215, FEC 2/3 in QPSK/DVB-S2 with PID 101.

Ensemble | IP-Address:Port
-- | --
SWR BW S (Baden Württemberg South, 8A and 8D) | 239.132.1.50:5004
SWR BW N (Baden Württemberg North, 9D) | 239.132.1.51:5004
Rheinland-Pfalz (11A) | 239.132.1.52:5004
Oberfranken (10B)  | 239.16.242.11:60011
Unterfranken (10A) | 239.16.242.13:60013
Bayern (11D) | 239.16.242.17:60017
Niederbayern (7D) | 239.16.242.16:60016
Obb/Schw (Oberbayern and Schwaben, 10A) | 239.16.242.15:60015
hr radio Hessen (7B) | 239.192.254.200:10000
  

### DAB-Ensembles (currently) not working

The format might be EDI, the reception is limited to only a few DVB-cards or professional equipment, as this is [DVB-GSE](https://www.dvb.org/standards/dvb-gse).

They are from Norway on 1.0ºW, 10719V, SR 4800, FEC 3/4 in DVB-S2/MIS=171 DVB-GSE 

Ensemble|
--|
NRK Reg2 BuTeVe |
NRK Reg3 SørRog |
NRK Reg4 HoSoFj |
NRK Reg5 HedOpp |
NRK Reg6 TrøMøRo |
NRK Reg7 NoTrFi | 


### Ensembles working with tsniv2ni

The format is ETI-NA(V.11)

Ensemble | Country | Sat | Freq | SR/FEC | Modulation | PID 
-- | -- | -- | -- | -- | -- | -- 
ERT DAB | Greece | 39ºE | 12242H | 13380 3/4 | QPSK/DVB-S2 | 1010
DAB Italia | Italy | 12.5ºW | 12518H | 2154 3/5 | QPSK/DVB-S2 | 777 
EuroDAB Italia | Italy | 12.5ºW | 12518H | 2154 3/5 | QPSK/DVB-S2 | 1025 
DAB Italia | Italy | 5ºW | 12690V | 29950 3/4 | 8PSK/DVB-S2 | 4105 
EuroDAB Italia | Italy | 5ºW | 12690V | 29950 3/4 | 8PSK/DVB-S2 | 4106 

## Guide

If you want to listen to one of these feeds, here's a guide how to do it (see below for some examples):

### dvbstream 

    dvbstream -f 12303000 -s 25546 8192 -p H -o | ts2na -s 12 -p 1063 | na2ni | ni2http --list
    
for UK's SDL National Mux or

    dvbstream -f 12223000 -s 13380 1010 -p H -o | tsniv2ni 1010 | ni2http --list
    
for the Greek Mux.

Please consider to add `-D x` (which stands for DiSEqC) if you have more than one LNB.

If you also have installed the fork https://github.com/satdreamgr/eti-tools then you can hear the German EDI streams even without setting up a DVB network connection with the new tool `fedi2eti`:

    dvbstream -f 12572000 -s 10215 8192 -p V -o | fedi2eti 101 239.16.242.17 60017 | dablin_gtk

for the Bayern Mux and output it to [dablin_gtk](https://github.com/Opendigitalradio/dablin).

### Alternative

- **Source**: You need to _capture_ the feed with a SAT tuner. Our recomendation is to use one of them to stream the feed to a multicast address. Then you can use this stream from any computer on your network (not only the one with the SAT tuner).
  - If your SAT tuner is a SAT>IP server, then you can use this URI for getting an MPEG-TS with the three DAB bitstreams present in the MUX of 4.5ºE:
    - `satip://server:554/?src=1&freq=12303&pol=h&msys=dvbs&mtype=qpsk&sr=25546&fec=78 &pids=0,1,16,17,18,20,1061,1062,1063,5060,5070,5080"`
- **Unpacking**: You need to _process_ the multicast stream to obtain the ETI DAB ensemble. You can do this on any computer on your network. The input must be the MPEG-TS and the output will be the ETI-NI bitstream. You can save it in a file, FIFO or PIPE. Required tools: `SOCAT`, `ts2na` & `na2ni`.
- **Play**: Finally you need to _reproduce_ the DAB ensemble. You can use any DAB player with ETI-NI input support (for example DABlin).

Here an "all-in-one" example:

- **Producer**: Using the DVBlast tool in a computer with a DVB-S tuner, for streaming all three DAB ensembles from 4.5ºE in 12303-H, to the multicast address udp://@239.1.1.1:1234 from the source address 192.168.1.33 (IP of this computer):
  - `dvblast -f 12303000 -s 25546000 -v 18 -S 1 -d "239.1.1.10:5018@192.168.1.33/udp 1 0 0,1,16,17,18,20,1061,1062,1063,5060,5070,5080"`
- **Consumer**: Use DABlin to consume the MPEG-TS from udp://@239.1.1.1:1234 and tune the ensemble "BBC DAB" at pid 1061:
  - `socat UDP4-RECV:5018,bind=239.1.1.10,ip-add-membership=239.1.1.10:eth0,reuseaddr - | ts2na -p 1061 -s 12 | na2ni | dablin -p`

