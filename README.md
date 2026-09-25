# Sonic Cipher

Single-file browser tool that encrypts short text with **AES-256-GCM** and sends it as sound (bundled [ggwave](https://github.com/ggerganov/ggwave)) or as copy-paste ciphertext. Nothing is installed and nothing is uploaded.

Open [`sonic-cipher.html`](sonic-cipher.html) in a browser on both devices.

## What is this for?

Two people who share a passphrase can send a short encrypted note as sound — phone speaker to phone mic — or as an `SC2.` text packet.

**Use cases**

- **Radio / ham / analog links** — play the tone into a transceiver or walkie-talkie; the far side holds a phone to the speaker and decrypts. Useful when you have voice but not a data channel.
- **Mobile phone calls** — put the call on speaker, play on one handset, listen on the other. Works across ordinary GSM/VoLTE voice, not only in the same room.
- **VoIP / video calls** — same idea over FaceTime, Signal, Zoom, Teams, or a radio app. The ciphertext rides the audio mix.
- **Air-gap / same room** — pass a note between laptops or phones with no cable, QR, or network.
- **Text packet** — encrypt to an `SC2.` string and send that over SMS, email, chat, or paper if sound is a bad path.

## Technical summary

- **Confidentiality / integrity:** AES-256-GCM via Web Crypto. Random 16-byte salt, 12-byte IV, 16-byte message id. Version, id, and timestamp are bound as AAD so headers cannot be swapped.
- **Key:** PBKDF2-HMAC-SHA-256, 600 000 iterations, from a shared passphrase (12+ characters). Strength tracks the passphrase, not a public-key identity.
- **Freshness:** millisecond timestamp; packets outside ±10 minutes are rejected. Duplicate ids on this page are logged but still shown.
- **Air layer:** bundled **ggwave** (FSK + Reed–Solomon). Audible and ultrasonic profiles at slow / medium / fast / ultrafast. Long payloads are split into 80-character frames and reassembled.
- **Compatibility:** legacy `SC1` blobs still decrypt. Current packets are `SC2`. Mic capture needs HTTPS or localhost; playback works from a saved file.

## What it runs on

- **iPhone / iPad** — Safari or any browser that can open a local HTML file or a hosted page. Grant the microphone when listening. Raise playback volume; iOS speakers are often quiet. FaceTime and cellular calls work if the far end can hear the tones.
- **Android** — Chrome, Firefox, or Samsung Internet. Same file. Allow the mic; turn off extra noise suppression in the browser if decode is flaky.
- **Desktop / laptop** — Chrome, Edge, Firefox, or Safari on Windows, macOS, or Linux. A real speaker and mic beat a Bluetooth headset for ggwave.
- **Other** — any device with a modern browser and Web Crypto + Web Audio (Chromium-based kiosks, Chromebooks). No native app and no install.

Both sides must use the same HTML revision, the same passphrase, and the same speed profile. Listen first, then play. Ultrasonic modes need speakers and mics that still respond above ~15 kHz — many phones do not.

## Two-device receive

1. Same passphrase and same profile on both devices.
2. On device B: Start listening and allow the mic.
3. Hold device A’s speaker about 10–40 cm from B’s mic (or play into a call / radio).
4. On device A: Encrypt & play.
5. Plaintext appears on B after a valid frame is decoded.

Microphone capture needs HTTPS or localhost. Encrypt, play, and text decrypt work from a saved `.html` file.

```bash
python3 -m http.server 8080
```
