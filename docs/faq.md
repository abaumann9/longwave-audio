# FAQ

## Can I use regular Wi-Fi instead of HaLow?

Yes. The snapclient firmware and Snapserver work over any IP network. If you would like to forgo the HaLow component of this project you can. But it's not as cool. You can also mix HaLow and standard Wi-Fi endpoints on the same Snapserver. As HaLow is native TCP/IP it all just works.


## Can I use this with sources other than Spotify?

Yes. Snapserver accepts audio from any source that can write to a named pipe or be captured as a stream. Common options:

- **Spotify** via librespot (Spotify Connect)
- **MPD** (Music Player Daemon) for local music libraries
- **AirPlay** via shairport-sync
- **Line-in** via ALSA loopback
- **Bluetooth** via bluealsa
- **Web streams** via any player that outputs to a pipe

Each source appears as a separate stream in Snapserver. Endpoints can be assigned to different streams.

## Do I need a HaLow access point?

Yes, on the server side you need a Wi-Fi HaLow AP to create the HaLow network. The endpoints connect to this AP. The AP can be connected to your server via Ethernet, USB, or other interface.

## Is this a finished product?

No. LongWave Audio is an early-stage open-source project. Hardware designs are being finalized and firmware is being adapted. It's suitable for builders and tinkerers who are comfortable with soldering, flashing firmware, and debugging — not a plug-and-play consumer product (yet).

## Where can I get help?

- Open an issue on the GitHub repository
- Post in relevant community forums (ESP32, Morse Micro, Snapcast)
