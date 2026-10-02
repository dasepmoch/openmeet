# Huddle

A bright, playful 3D meeting space for the web: walk around as an avatar, grab a seat, throw a
paper ball at a coworker and watch shared screens on a huge in-world wall.

Rooms are multiplayer: pick a name in the lobby, then create a room or join one by its code
(share the invite link). Talk with spatial voice and put your screen on the wall for everyone.
You can also play offline with seven seeded AI office-mates.

## Run in 3 commands

```sh
corepack enable        # or install pnpm 11
pnpm install
pnpm dev               # web on http://localhost:5173, server on :8787
```

Requires Node 24 LTS or newer and a WebGL2-capable browser. Open the page in two windows and join
the same room code to see each other.

Production: `pnpm build`, then `HUDDLE_STATIC_DIR="$PWD/apps/web/dist" pnpm start` serves the
client and the realtime endpoint (`/ws`) from one port (8787). Server settings are environment
variables, see `apps/server/src/config.ts`. Browsers only allow the microphone and screen capture
on HTTPS (or `localhost`), so put the server behind a TLS proxy (`HUDDLE_TRUST_PROXY=true`).

### Voice and screen sharing in production

Media goes peer to peer between up to `HUDDLE_MESH_CAP` people per room (default 8). People on
different networks usually need a TURN relay; Huddle hands out short-lived credentials for
[coturn](https://github.com/coturn/coturn) (sample config in `deploy/coturn/turnserver.conf`):

| Variable                  | Meaning                                                               |
| ------------------------- | --------------------------------------------------------------------- |
| `HUDDLE_ICE_URLS`         | Comma-separated `stun:` / `turn:` / `turns:` URLs (no public default) |
| `HUDDLE_TURN_SECRET`      | coturn `static-auth-secret`; without it TURN URLs are not handed out  |
| `HUDDLE_TURN_TTL_SECONDS` | Credential lifetime (default 3600, refreshed at half)                 |
| `HUDDLE_MESH_CAP`         | People with voice and screen per room (2 to 16, default 8)            |

```sh
HUDDLE_ICE_URLS="stun:turn.example.com:3478,turn:turn.example.com:3478?transport=udp,turns:turn.example.com:5349" \
HUDDLE_TURN_SECRET="$(cat /run/secrets/turn)" HUDDLE_STATIC_DIR="$PWD/apps/web/dist" pnpm start
```

## Scripts

| Command                     | What it does                                       |
| --------------------------- | -------------------------------------------------- |
| `pnpm dev`                  | Vite dev server + realtime server with watch       |
| `pnpm build`                | Production build of every package                  |
| `pnpm typecheck`            | `tsc` per package                                  |
| `pnpm test`                 | Vitest (engine, shared, content, avatar sculpting) |
| `pnpm lint` / `pnpm format` | ESLint / Prettier (`pnpm format:check` in CI)      |
| `pnpm e2e`                  | Playwright: multiplayer, play, voice and sharing   |
| `pnpm bots`                 | Load test: fake clients joining one room           |

`pnpm test` includes the server (rooms, seats, chat limits, resume, signaling, presenter rules,
TURN credentials, throws, hits and baskets), WebRTC negotiation against a fake peer connection, a
50-client swarm over real WebSockets, and throwing soaks: projectiles stay within every cap
(`HUDDLE_SOAK_MINUTES=30` for half an hour of simulated mischief) and splat decals within budget
through a 30-minute storm. `pnpm e2e` (build first) starts the server on port 8790 with the
built client; set
`PW_CHANNEL=chrome` to use an installed Chrome instead of Playwright's Chromium. Without a GPU
(CI) it runs on SwiftShader with `?render=0`. The voice test uses Chrome's fake media devices: a
generated voice-like WAV as the microphone and an auto-picked screen (a canvas stream where a
headless machine has no screen to capture).

Visual tours for self-review (dev server running):
`node scripts/visual-social.mjs [baseUrl] [outDir]` (bots, throwing, chat, board, theater, editor)
and `node scripts/contact-sheet.mjs out.png 2 a.png b.png ...`.

## What you can do

- Create or join a room by code (optional passcode); copy the invite link from the room code in
  the top bar. Everyone sees each other walk, sit, chat, emote, raise hands and throw things.
- The host (first one in) can switch the room mode (Serious: no throwing and only quiet
  gestures; Casual: everything; Party: everything plus party lights, sparkle trails and
  confetti), lock throwing, mop the room clean for everyone, reset the hoop scores, lower hands,
  hand over the host role or remove someone. Dropped connections come back to the same seat
  within 45 seconds.
- Walk, run, jump; click the floor to walk there, click a chair to sit.
- Throw things (`1`-`6`, then click): paper ball, paper plane, tomato, rubber duck, confetti and
  hearts. Tomatoes splat on walls (for a minute) and on people (who wipe their face). Paper
  planes glide; tilt their wings with the mouse wheel while aiming to curve the flight, and pick
  landed ones up again with `E`. Bots throw back when hit.
- Shoot hoops at the basket on the east wall: paper balls and ducks score 2 points, 3 from far
  away, on the scoreboard next to it.
- "Don't hit me" in the dock: thrown things pass right through you (and you don't throw).
  Everything is capped: a few things in the air per person, a limit per person and per room.
- Hold things: a coffee mug or water (from the machines, `E`), a pizza slice from the bar, a
  phone or a sign (+1, ?, BRB, ♥) from the emote wheel. `X` uses it (sip, bite, show the phone,
  raise the sign).
- Emote wheel (`Q`), raise your hand (`H`, with a queue in the People panel), chat (`Enter`) with
  speech bubbles over heads.
- Talk: test your microphone in the lobby (you join muted), then `M` to unmute, hold `V` to push
  to talk, `N` to deafen. Voices come from the avatars (spatial audio); name tags light up while
  someone speaks. Per-person volume in the People panel, devices and processing in Settings. The
  host picks the voice mode (Zones, Everyone, Nearby) and can mute everyone.
- Share your screen (dock button, or `E` at the lectern): it appears on everyone's wall, with its
  sound coming from the screen. One presenter at a time: others can ask you to hand over, the host
  can take over. Mark the content as Text (sharp) or Motion (smooth) in the presenter bar.
- Click the screen or press `T` for Theater Mode (zoom with the wheel, drag to pan, volume,
  fullscreen); `Esc` goes back to your seat. Draw on the whiteboard.
- Light scenes (Meeting, Present, Relax, Party), an avatar editor, quality presets with an
  automatic scaler, synthesized sound (footsteps, a quiet room tone, swishes and splats).

## Controls

| Input                   | Action                                           |
| ----------------------- | ------------------------------------------------ |
| `W A S D` / arrows      | Walk (`Shift` runs, `Space` jumps)               |
| Left drag / right drag  | Look around; mouse wheel zooms into first person |
| `Shift` + drag / middle | Slide the camera                                 |
| Click                   | Walk, sit, open the whiteboard, or throw         |
| `E`                     | Sit / stand, coffee, water, pizza, pick up       |
| `1`-`6`, `G`            | Pick something to throw, quick paper ball        |
| Wheel while aiming      | Tilt a paper plane's wings                       |
| `X`                     | Use what is in your hand                         |
| `Q`, `H`, `Enter`       | Emotes and things to hold, raise hand, chat      |
| `M`, hold `V`, `N`      | Mute, push to talk, deafen                       |
| `C`, `R`, `F`, `T`      | Next camera, recenter, look at screen, theater   |
| `Esc`                   | Close or cancel                                  |

Keys can be remapped in Settings.

## URL options

| Parameter                        | Effect                                                    |
| -------------------------------- | --------------------------------------------------------- |
| `?room=CODE`                     | Prefills the lobby (invite links look like this)          |
| `?autojoin=1&room=CODE&name=Ana` | Joins right away                                          |
| `?offline=1`                     | Skips the lobby: offline with AI office-mates             |
| `?bots=N`                        | Offline, with N office-mates (0-12, default 7)            |
| `?screen=idle`                   | Shows the clock on the wall instead of the demo deck      |
| `?debug=1`                       | Exposes `window.__huddle` hooks in production builds      |
| `?render=0`                      | Simulation and networking without drawing (tests, no GPU) |

## Docs

[Architecture](docs/ARCHITECTURE.md) · [Protocol](docs/PROTOCOL.md) · [Porting](docs/PORTING.md) ·
[Content](docs/CONTENT.md) · [Third-party licenses](THIRD_PARTY.md)
