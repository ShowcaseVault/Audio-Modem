# Airwave: theory of operation

Airwave moves text between two devices using only a speaker and a
microphone. This document explains *why* the codec is built the way it is —
the physical constraints of an acoustic channel, and the specific
techniques chosen to survive them.

## The channel is hostile in a specific way

A room is not a wire. Three effects dominate:

- **Noise.** Fans, traffic, speech — broadband and unpredictable.
- **Reverberation.** A speaker's sound arrives at the microphone many times,
  once directly and then again and again off walls, each copy delayed and
  attenuated. This is the dominant failure mode, not noise.
- **Clock mismatch.** The transmitting device's DAC and the receiving
  device's ADC run on independent clocks. Over the length of a message,
  their sample rates drift apart.

Everything in Airwave's design is a response to one of these three.

## Modulation: 4-FSK

Airwave encodes two bits per symbol as one of four tones (4-ary frequency-
shift keying). Four frequencies are chosen per "band," spaced widely enough
that a Goertzel filter (a narrowband DFT bin, computed directly rather than
via a full FFT) can tell them apart even with a mistuned symbol clock or a
few dB of channel droop between them.

FSK was chosen over amplitude or phase modulation because:

- **Amplitude (ASK/OOK)** is wrecked by the automatic gain control in cheap
  microphones and by the room's own frequency response — a null at the
  carrier frequency looks identical to a "0" bit.
- **Phase (PSK)** requires the receiver to track a stable phase reference
  across the message. Reverberation scrambles phase far more violently than
  it scrambles frequency, since each echo arrives with its own delay-
  dependent phase shift.
- **Frequency** survives both. A tone's frequency doesn't change when it
  echoes, only its amplitude and phase — and the receiver only cares which
  of four frequencies had the most energy in a window, which per-tone
  amplitude changes rarely reverse.

Two bits per symbol (rather than one, i.e. plain FSK) roughly doubles
throughput for the same symbol duration, at the cost of needing four
distinguishable tones instead of two. Going beyond four widens the tone
spacing needed for reliable separation, in a chunk of spectrum small enough
to fit under 20 kHz — the practical ceiling above which consumer speakers
and microphones stop responding.

**Gray coding.** The four tones are not assigned bit-pairs in numeric order.
Adjacent tones in frequency are assigned bit-pairs that differ in only one
bit (`00, 01, 11, 10` — a Gray code). If a mistuned or noisy detector picks
the tone next to the correct one, the error costs exactly one bit, not two.
This matters because that single-bit error is exactly the kind Hamming(7,4)
below is built to fix for free.

## Three bands, three symbol rates

Airwave defines three frequency bands (low ~2.6–6.5 kHz, mid ~9.6–13.5 kHz,
high ~15.2–19.1 kHz) and three symbol durations (40 ms, 20 ms, 10 ms). This
is a 3×3 design space, not an arbitrary pile of options:

- **Band** trades audibility for hardware rolloff. Low-band tones are
  clearly audible but nearly every speaker/mic reproduces them faithfully.
  High-band tones are at or past the edge of what many consumer devices
  push out or pick up, but they're inaudible to most adult hearing.
- **Symbol length** trades throughput for echo tolerance. A room's echo
  tail is fixed by its geometry — a symbol has to be long enough that the
  tail from one symbol decays before the next one is judged, or energy
  from symbol *N* smears into the measurement for symbol *N+1*
  (inter-symbol interference). Longer symbols dodge this at the cost of
  fewer bits per second; shorter symbols only work in a dry, close-range
  room.

The "Check this device" feature measures which band survives the local
speaker-to-microphone path best, since consumer audio hardware varies more
in frequency response than the underlying codec's tolerances do.

## Nine sync tones: self-describing frames

Each of the 3 bands has 3 dedicated sync tones — one per symbol rate — for
9 total. A frame opens with a sync burst on exactly one of these nine
tones. Because each tone maps to a unique (band, rate) pair, hearing which
tone the sync burst used tells the receiver both facts at once, without
either device needing to agree on settings beforehand. This is why the two
ends of an Airwave link don't need matching configuration — the receiver
scans all nine tones simultaneously and adapts to whatever it hears.

Before the sync burst, a short carrier preamble (eight symbols alternating
between the two outer data tones) gives the receiver's gain-tracking time
to settle before anything meaningful arrives — the first fraction of a
second of any capture is typically distorted by AGC/analyser warm-up.

## Recovering the symbol clock without a shared reference

The two devices share no clock. Worse, echo can make a naive "find the
rising edge" approach lock onto a reflection instead of the direct path,
which is wrong by a fraction of a symbol *or* by several whole symbols.

Airwave's receiver doesn't trust any single edge. Instead, once a sync tone
is detected, it:

1. Tries a spread of candidate start times across the sync burst.
2. For each candidate, demodulates and scores confidence as the mean gap
   (in dB) between the winning tone's energy and the runner-up's — a score
   that peaks sharply exactly at the correct symbol boundary, and falls off
   fast on either side.
3. Ranks candidates by that score, then — starting from the best — decodes
   the header and checks it against its own checksum.
4. Keeps the first candidate whose header check passes.

The insight: multipath can absolutely shift where an *energy* edge appears,
but it essentially cannot forge a header that passes an 8-bit checksum by
accident. Verifying against the header, not just against signal strength,
is what lets the receiver ignore a confident-looking but wrong echo lock.

## Error correction: Hamming(7,4)

Every 4-bit nibble of real data is expanded to a 7-bit Hamming codeword
before transmission. Hamming(7,4) can *correct* any single-bit error and
*detect* (though not correct) two-bit errors within a codeword, using 3
bits of parity per 4 bits of data — a fixed, small overhead in exchange for
tolerating the single-bit-flip errors that Gray coding was chosen to
produce in the first place.

The header (frame length + checksum) is small enough that Airwave affords
extra redundancy: it's sent three times, and the receiver majority-votes
the three copies bit-by-bit before Hamming-decoding the result. Getting the
header right matters disproportionately, since it tells the receiver how
many bytes of body to expect.

## Interleaving: spreading out burst damage

Hamming(7,4) is strong against isolated bit errors but collapses if *two*
bits inside the same 7-bit codeword are wrong — which is exactly what a
short noise burst or a deep echo null tends to produce, since it corrupts
several consecutive symbols at once.

Airwave's body is block-interleaved: instead of transmitting codeword 1 in
full, then codeword 2, then codeword 3, it transmits bit 1 of every
codeword, then bit 2 of every codeword, and so on. A contiguous burst of
symbol errors on the wire is now spread across many codewords, damaging at
most one bit in each — squarely inside what Hamming(7,4) can fix — rather
than obliterating one codeword outright.

## CRC-16: the last line of defense

Hamming(7,4) is corrected on a best-effort basis; it can silently miscorrect
if two bits in a codeword happen to fail in exactly the pattern that looks
like a valid single-bit error elsewhere. A CRC-16 checksum is appended to
the payload before nibble-splitting, so that even if every codeword
"corrects" without complaint, the final decoded bytes are verified against
a checksum before being shown to the user. A message that fails the CRC
after correction is reported as failed, not delivered with silently wrong
content — Airwave never surfaces a message that hasn't been positively
verified.

## Frame layout, end to end

    carrier   8 symbols, alternating tones -> lets receiver gain settle
    sync      4 symbols, 1-of-9 tones      -> identifies band + symbol rate
    header    length + checksum, 4x Hamming(7,4), sent 3x, majority-voted
    body      payload + CRC-16 -> nibbles -> Hamming(7,4) -> interleaved

Every layer exists because of a specific, named failure mode: FSK for
multipath-proof detection, Gray coding to make tone-confusion single-bit-
cheap, Hamming to fix that single bit, interleaving to keep burst errors
from ever exceeding one bad bit per codeword, majority-voting to protect
the header specifically, and CRC-16 as a hard backstop that catches
whatever slips through everything else uncorrected.
