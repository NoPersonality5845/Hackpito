# Hackpito: Google Home speakers → local server + Claude + an Ultron-style voice

Research date: November 2026

## The short answer

**You can't reflash a stock Google Home to point at your own server.** The firmware is signed, the
bootloader is locked, and as of late 2026 there is no public root or custom firmware for any Google
Home or Nest speaker. Google is still pushing official firmware (3.79 is in preview right now), and
**Gemini for Home** is replacing Google Assistant on these speakers. Once a home switches to Gemini,
it can't switch back. No setting exists to redirect the assistant to another backend.

That leaves two routes that work, and you can do them in order:

| Route | What you do | Effort | Result |
|---|---|---|---|
| **A: Ears + speaker (start here)** | Leave the Google Homes stock but **flip their mic-mute switch**. Put a Home Assistant voice device (for example the **Home Assistant Voice Preview Edition**, about $59) next to each one. The voice device hears you. Claude answers. Your server casts the reply and the music to the Google Home. | One weekend | Everything you asked for, working, without opening anything |
| **B: Brain transplant** | Open the speaker and **replace its mainboard** with an open ESP32 board (Onju Voice, iMike78, MiciMike). It becomes a Home Assistant voice satellite inside the original shell. | 4 to 8 weeks (mostly waiting for boards) | No Google code left in the device at all |

Muted Google Homes still work as Cast speakers, because casting doesn't use the microphones. That's
why Route A works without hacking anything.

## Architecture (both routes)

```
 you speak ──► mic device (Voice PE, or the replaced board in Route B)
                 │  wake word on the device ("Hey Ultron": custom microWakeWord)
                 ▼
        Home Assistant server (on your LAN)
          ├─ Speech-to-text: Whisper (local)
          ├─ Conversation agent: Anthropic integration ──► Claude API (cloud)
          ├─ Text-to-speech: Piper (local) + "Ultron" effects chain
          └─ Music Assistant ── Spotify provider (needs Spotify Premium)
                 │
                 ▼  Google Cast
        Google Home #1  ◄── transfer queue / cast group ──►  Google Home #2
```

How each requirement is covered:

- **Local server instead of Google:** Home Assistant runs the whole pipeline. The one part that
  leaves your network is the text sent to Claude. Claude is a cloud API, so "local" here means local
  except for the language model.
- **Claude answers:** the official *Anthropic Conversation* integration. You paste in an API key from
  console.anthropic.com. It can answer general questions and control any entities you expose to it,
  and it has an optional web-search tool. A fast, cheap model like Haiku typically costs a few
  dollars a month for household use.
- **Spotify:** Music Assistant's Spotify provider is the music source, and the Google Homes are Cast
  players. You can also add the Spotify Connect plugin so the speakers show up in the Spotify app.
  Spotify Premium is required.
- **Passing music between speakers:** Music Assistant can move the play queue from one player to
  another. For both rooms at once, create a speaker group in the Google Home app. Music Assistant
  plays to it with tight sync. You could then add a script so Claude can do this by voice, for
  example "Ultron, move the music to the bedroom."
- **Ultron voice:** see the section below.

## About the Ultron voice

Ultron's film voice belongs to a real actor (James Spader). Training a clone on movie audio means
copying a real person's voice without their consent, so this plan doesn't do that. The voice is also
far less recognizable from the voice itself than from the processing on it. The practical approach is
a **sound-alike effects chain**:

1. Start from a deep, calm male Piper voice. Try several of the English voices and pick the one with
   the best cadence.
2. Run its output through effects: pitch down slightly, add a subtle metallic doubling or flanger and
   light distortion, then a short reverb. A starting point with `sox`:
   ```
   sox in.wav out.wav pitch -250 overdrive 4 10 flanger 0 2 0 71 0.5 sine 25 lin reverb 25 50 40 equalizer 120 1q +4
   ```
3. Wrap Piper plus `sox` as a small custom Wyoming TTS service so Home Assistant uses it like any
   other voice.
4. Give Claude the personality through a system prompt in the conversation agent settings: measured,
   sardonic, theatrical, but still helpful and brief.
5. Optionally, train a custom **"Hey Ultron"** wake word with microWakeWord. This takes an evening
   or two.

## Real timeline

| Phase | Work | Calendar time |
|---|---|---|
| 0. Hardware | Choose a server (Raspberry Pi 5 with 8 GB, or a used N100 mini PC, which is much better for Whisper) and order one Voice PE per room | Shipping, about 1 week |
| 1. Server | Install Home Assistant OS, add Whisper, Piper, Music Assistant, Spotify, Cast | 2 to 4 hours |
| 2. Claude | Add the Anthropic integration, write the persona prompt, expose entities, set the voice pipeline | 1 to 2 hours |
| 3. First working version | Voice PE → Claude → reply on the Google Home, Spotify playback and transfer between speakers | **End of weekend 1** |
| 4. Ultron voice | Wrapper for Piper plus effects, tuning by ear | 1 to 3 evenings |
| 5. Wake word | Train "Hey Ultron" with microWakeWord | 1 to 2 evenings |
| 6. Polish | Music scripts for voice ("move it to…", "play X everywhere"), audio ducking, reliability | Ongoing, 1 to 2 weeks |
| 7. *(Optional)* Route B | Order or assemble replacement boards, solder or flash over USB, fit them in the shells | **+4 to 8 weeks**, prototype-quality hardware |

**Realistically:** about two weeks to a solid version with the Ultron voice on Route A. Add one to
two months if you want the Google hardware fully converted with Route B.

## Route B hardware notes

What you can do depends on **which model** you own:

- **Nest Mini (2nd gen):** best supported. Options are Onju Voice (ESP32-S3; you order the PCB
  yourself and there's no finished product), the iMike78 drop-in PCB (ESP32-S3/P4) and the MiciMike
  board (ESP32-S3 plus an XMOS chip like the Voice PE). Community ESPHome firmware exists. The first
  flash must be over USB before the board goes in the shell.
- **Home Mini (1st gen):** the MiciMike Home Mini board is crowdfunding or prototype stage.
- **Original Google Home (2016), Nest Audio:** no drop-in board known. Stay on Route A, or gut the
  shell and fit an ESP32-S3 board plus amplifier yourself (custom project).

## Shopping list for Route A

- A server: Raspberry Pi 5 with 8 GB and an SSD, or an Intel N100 mini PC (about $150 to $200)
- Home Assistant Voice Preview Edition × 2 (one per Google Home)
- An Anthropic API key, with a monthly spend limit set in the console
- Spotify Premium (you likely have it already)

## Next steps

1. Tell me the exact model of each Google speaker (look at the bottom label or in the Home app under
   Device information).
2. Pick the server hardware.
3. I'll write the code that lives in this repo: the Wyoming TTS wrapper for the Ultron voice, the
   Claude persona prompt, the Music Assistant scripts that Claude can call by voice, and the setup
   steps.

## Sources

- [Old Google Home and Nest speakers getting big firmware update (9to5Google, Oct 2026)](https://9to5google.com/2026/10/14/google-home-nest-speaker-update/)
- [Gemini for Home replaces Google Assistant (The Gadgeteer)](https://the-gadgeteer.com/2026/06/23/gemini-for-home-replaces-google-assistant/)
- [Home Assistant Voice PE](https://www.home-assistant.io/voice-pe)
- [Anthropic Conversation integration](https://home-assistant.io/integrations/anthropic)
- [Music Assistant: Google Cast](https://music-assistant.io/player-support/google-cast)
- [Music Assistant 2.4: Spotify Connect](https://alternativeto.net/news/2025/3/music-assistant-2-4-brings-assist-devices-enhancements-spotify-connect-and-podcast-support/)
- [Onju Voice](https://github.com/justLV/onju-voice) · [onju-voice-esphome](https://github.com/Idskov/onju-voice-esphome)
- [iMike78 nest-mini-drop-in-pcb](https://github.com/iMike78/nest-mini-drop-in-pcb)
- [MiciMike Google Home Mini PCB (CNX Software, Apr 2026)](https://www.cnx-software.com/2026/04/29/micimike-open-source-drop-in-pcb-converts-google-home-mini-into-a-local-voice-assistant/)
