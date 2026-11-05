# Sonic Cipher

Single-file browser tool that sends a short message over **any voice path** — HF
amateur SSB, FM radio and walkie-talkies, landlines, VoIP, or a phone held up to
a speaker — using a narrowband MFSK modem built specifically for those channels.
Optionally encrypted with AES-256-GCM, with X25519 key exchange.

- **Hosted (HTTPS, so the microphone works):** https://markdaman1236.github.io/Sonic-Cipher/
- **Or download `index.html`** and open it locally. Nothing is installed, nothing
  is uploaded, and the page makes no network requests at all.

On iPhone, open the hosted page and use **Share → Add to Home Screen**. A local
file loses microphone access on iOS (and possibly Web Crypto); the page reports
what is available on your device in a panel at the top.

---

## ⚠️ Amateur radio and encryption

In most jurisdictions — FCC §97.113(a)(4) in the US and the equivalent rules
across ITU Regions 1–3 — transmitting messages **encoded to obscure their
meaning is not permitted on amateur bands**.

Use **Clear text** mode on the air. It is an ordinary digital mode and keeps the
full error correction, so it is just as robust. Keep the encrypted modes for
wireline paths (landline, VoIP) and for licences that do permit encryption.

---

## Use it

1. **Receiver first.** On the receiving device press *Start listening* and allow
   the microphone. It auto-detects the profile, so only the sender chooses one.
2. **Pick a mode** — Passphrase, Key exchange, or Clear text.
3. **Pick a channel profile** on the sending device and press *Transmit*. The
   page shows how long the transmission will take before you start.
4. On a radio, key up, wait for the tone to begin, and **hold PTT for the whole
   transmission including the gaps**.

### Channel profiles

| Profile | Baud | Tones | Use for |
|---|---|---|---|
| `hf-robust` | 15.625 | 1200–1669 Hz | Weak HF SSB, and through voice codecs (AMR/GSM/Opus) |
| `hf-normal` | 31.25 | 1000–1938 Hz | Ordinary HF SSB |
| `fm` | 62.5 | 700–2575 Hz | VHF/UHF FM, walkie-talkie |
| `wire` | 125 | 700–2575 Hz | Landline, VoIP, same room |

Start with `wire` for a same-room or landline test — it is the fastest. Use
`hf-robust` through a phone or VoIP call, since it is the one that survives
speech codecs.

### WAV in and out

*Download WAV* writes a 48 kHz mono file you can feed straight into a rig through
a soundcard interface. *Decode a WAV…* reads one back. This is the easiest way to
bench-test without two live devices — and on iOS it lets you receive without
microphone permission at all: record the tones with Voice Memos, then decode the
file.

### Key exchange

Each side sends a one-time X25519 public key, then both display the same four
words. **Read those words aloud over the voice channel you already have.** If
they match on both ends, nobody is in the middle. If you also set a passphrase,
the exchange is authenticated by it and the words become a double check.

---

## How it works

16-tone MFSK, every tone inside **700–2600 Hz**, with one tone sounding at a time
at constant amplitude.

- **Two-tone probe** measures the channel's frequency offset before any data is
  demodulated, so an SSB rig mistuned by a few hundred hertz still decodes.
- **Goertzel detection** evaluates arbitrary frequencies rather than fixed FFT
  bins, so the measured offset is simply applied.
- **Lead-in warble** keys VOX, settles AGC and converges echo cancellers and
  jitter buffers. It alternates tones rather than holding one, because a steady
  tone is what a network echo canceller learns to suppress.
- **Cauchy MDS erasure coding** over GF(256) with a per-block CRC: the message is
  split into small blocks and any sufficient subset rebuilds it. The sender keeps
  emitting fresh parity blocks and the receiver finishes as soon as enough
  arrive — no back-channel, which is what one-way radio requires.
- **Preamble and sync repeat before every block group**, so a burst landing on
  one acquisition point does not cost the message.
- **Constant envelope** means nothing to clip, constant FM deviation, and no
  amplitude for a limiter or compressor to chew on.

### Crypto

- **Passphrase:** PBKDF2-HMAC-SHA-256 (600 000 iterations, derived once and
  cached) → HKDF per message → AES-256-GCM. The envelope is an 8-byte nonce plus
  ciphertext and tag — 52 bytes for a 28-character message.
- **Key exchange:** X25519 (32-byte keys; P-256 fallback) with HKDF, and a
  4-word, 24-bit short authentication string for voice verification.
- **Passphrase + key exchange together** gives an authenticated exchange with
  forward secrecy: the passphrase becomes the HKDF salt, so an active attacker
  cannot derive the session even if the words are never compared.
- **Replay protection** remembers nonces. It does **not** depend on clocks
  agreeing between the two devices.

### Measured

Tested against a channel model including band-limiting, noise, SSB tuning error,
Watterson selective fading, clock drift, clipping, µ-law companding, dropouts and
impulsive noise — and separately through real G.711, GSM 06.10, AMR, Opus, G.722,
iLBC and Speex encoders.

- Clean links, good and mistuned SSB, good and marginal FM, and landlines
  (including a poor one): **decodes every time**.
- Marginal HF SSB at 6 dB SNR with CCIR-poor fading: reliable on the HF profiles.
- **Through every speech codec tested**, including AMR at its lowest 4.75 kbit/s
  rate: the HF profiles decoded every attempt. A mobile or VoIP *voice call* is a
  workable path.

**Limits, honestly:** at 0 dB SNR *combined with* severe HF fading it fails —
either impairment alone is fine, and fixing the combination needs a
soft-decision code of the kind Olivia uses, not more parity. And a link dropping
out more than roughly once a second in long bursts defeats it, though by then
voice on the same link is badly broken up too.

---

## What changed from the previous version

The old audio layer was [ggwave](https://github.com/ggerganov/ggwave), which
could not work on these channels — not as a tuning problem, but physically. It
places tones on an FFT bin grid, and even its narrowest modes occupied roughly
**1125–4125 Hz**, while a landline passes 300–3400 Hz and a walkie-talkie or SSB
rig passes about 300–2700 Hz. The upper half of every byte was filtered away
before it arrived. Its sync markers also used sixteen simultaneous tones, which
any FM limiter, phone companding or audio compressor turns to mush, and it had no
way to measure the frequency error SSB tuning always introduces. A 28-character
message also became 176 characters on air, with no retransmission and no
cross-frame FEC, so losing one frame of eight destroyed the whole message.

Also fixed along the way:

- Playback had **three gain stages in series** and clipped hard at the raised
  volume the old help text recommended. Now one stage that cannot exceed full
  scale.
- Capture used a deprecated main-thread node that **dropped audio** whenever the
  page was busy — and it was busy, logging and running PBKDF2. Now an
  AudioWorklet.
- The receiver compared each decode with the previous one and returned early,
  which **silently discarded a deliberate resend**. Repeats are now suppressed
  only within a single transmission, so resending always works.
- Decryption **rejected anything more than ten minutes from your clock**, so two
  devices with disagreeing clocks failed every message.
- PBKDF2 ran its 600 000 iterations on *every* decrypt attempt.

Packets from the older version (`SC1.` / `SC2.`) still decrypt in the text panel.

---

## Requirements

Any current browser with Web Audio and Web Crypto — iOS Safari, Android Chrome or
Firefox, and desktop Chrome, Edge, Firefox or Safari.

Microphone capture needs **HTTPS or localhost**. Transmitting, WAV export/import
and text encryption all work from a saved file. For radio work a cable into the
rig beats acoustic coupling, and on a phone check the ring/silent switch (it
mutes Web Audio in Safari) and set Auto-Lock to Never so the screen does not
sleep mid-transmission.
