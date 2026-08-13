# Airwave

Send text messages between two devices using nothing but a speaker and a
microphone. No radio, no network, no pairing.

Four audio tones carry two bits each (4-FSK). Every nibble is protected by a
Hamming(7,4) code and the whole body is block-interleaved, so a burst of noise
damages many codewords slightly instead of one codeword fatally. A CRC-16
catches whatever the error correction cannot repair. Messages over 32 bytes
are split into independent chunks, each with its own CRC, so a bad chunk
doesn't take the rest of the message down and text renders as chunks arrive.

For the full theory behind these choices, see [airwave.md](airwave.md).

**Live demo:** [airwave.vishalsigdel.com.np](https://airwave.vishalsigdel.com.np)

## Deployment

Served via GitHub Pages on a custom domain. The `CNAME` file at the repo root
pins the domain to `airwave.vishalsigdel.com.np`; point a DNS `CNAME` record
for that host at `<github-username>.github.io`, then enable Pages for the
`main` branch in the repo settings.

## Running it

Microphone access requires a secure origin. GitHub Pages works; so does
`http://localhost`. A plain LAN address like `http://192.168.1.20:8000` will be
refused by the browser.

Locally:

    python3 -m http.server 8000
    # open http://localhost:8000

Transmitting needs no permission at all, so a device on an insecure origin can
still send to a device that is properly served.

## Using it

1. **Check the channel first.** Press "Check this device" — it sweeps the
   speaker and measures the response on the microphone, then recommends the
   band with the best margin. Do this on both devices; the weaker of the two
   measurements is what limits the link.
2. **Start listening** on the receiver.
3. **Send** from the other device.

Band and symbol length are detected automatically on receive — nine distinct
sync tones exist, one per band-and-rate combination, so hearing one identifies
both. The two devices do not need matching settings.

## Choosing a symbol length

Measured against a simulated channel with additive noise, sample-clock
mismatch and a specular room echo:

| Symbol length | Payload rate | Holds up when |
|---|---|---|
| 40 ms | ~27 bps | Almost anything, including a very live room |
| 20 ms | ~54 bps | Echo delay shorter than one symbol |
| 10 ms | ~108 bps | Quiet, fairly dry room, short distance |

Noise is rarely the limit — narrowband detection over a whole symbol gives
enough processing gain that frames survive well below 0 dB wideband SNR.
Reverberation is the real limit, and longer symbols are the answer.

If a link refuses to work at one setting, change the symbol length. A room
reflection can put a cancellation null right on one sync tone, and moving to
another rate shifts that tone elsewhere.

## Frame layout

    carrier   8 symbols, outer two tones alternating (lets AGC settle)
    sync      4 symbols, one of nine tones -> identifies band + symbol length
    header    msgId+chunkIdx, chunkTotal, length, check -> 8x Hamming(7,4), sent 3x, majority voted
    body      chunk payload (<=32 bytes) + CRC-16 -> nibbles -> Hamming(7,4) -> interleaved

...repeated once per chunk for messages longer than 32 bytes, up to 16 chunks
(512 bytes) per message.

Symbols are Gray-coded across the four tones, so mistaking a tone for its
neighbour costs one bit rather than two — exactly the error Hamming repairs.

Rather than trusting a measured signal edge for the symbol clock, the receiver
tries a spread of offsets across the whole sync burst, ranks them by how
confidently the four tones separate, and keeps the first whose header check
passes. Multipath can shift an apparent edge by whole symbols; it cannot forge
a valid header.
