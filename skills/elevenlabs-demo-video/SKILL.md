---
name: elevenlabs-demo-video
description: Build a narrated product-demo / walkthrough video of a running web app — Playwright screen-records a guided tour, ElevenLabs synthesizes the voiceover, and ffmpeg lays the narration over a soft music bed (ducked under the voice). Use when someone wants an AI-voiced demo video, a screencast with voiceover, a product walkthrough/reel, or to re-record an existing demo after UI changes.
---

# ElevenLabs-powered demo videos

A repeatable recipe for turning a running web app into a polished, narrated demo
`mp4`. One self-contained Node script per demo does everything: synthesize the
voiceover, screen-record a choreographed tour of the app, and mux voice + music
under the footage. No video editor, no manual timing.

## The pipeline

```
ElevenLabs TTS ──► narration .wav per line (cached)
                                   │
Playwright  ──► record a scripted scroll/click tour of the live app (.webm)
                                   │
ffmpeg      ──► trim the load-in lead, place each narration line at its scene
               cue, generate an ambient music bed, duck the music under the
               voice (sidechain compress), encode the final .mp4
```

The key idea: **each scene holds exactly as long as its narration line**, so the
voice always lands on the right frame without hand-syncing.

## Prerequisites

- Node 18+ and `npm i -D playwright` (uses the installed Chrome via
  `channel: "chrome"`, or swap to bundled chromium).
- `ffmpeg` and `ffprobe` on `PATH`.
- An ElevenLabs API key with **text-to-speech** permission (the cheaper keys often
  lack `user_read`, so don't depend on the quota endpoint — just try TTS).
- The app running locally (e.g. `npm run dev` on `:3000`); pass `BASE_URL` to override.

Store the key outside git (e.g. a gitignored `demo/.eleven.key`). The rendered
`.mp4` files are large — gitignore them and distribute via a share (Drive, etc.),
not the repo.

## How to use it

1. **One script per demo.** Copy the template below to `demo/build-<name>.mjs`.
   Each script is independent and writes its scratch files to `demo/<name>-work/`.
2. **Write the narration** as the `LINES` array — verbatim, one entry per scene.
   Aim ~2.3 words/sec when budgeting time.
3. **Choreograph the scenes** in `actions` — one entry per line, in the same order.
   A plain async fn composes a still shot; an `{action, hold}` object composes the
   shot (`action`) then pans/clicks **while the narration plays** (`hold`).
4. **Run it in the FOREGROUND** for any demo that drives the app (clicks, form
   fills, identity/route switches): `node demo/build-<name>.mjs`. Static
   scroll-only tours can run backgrounded, but interactive ones often can't reach
   the dev server or drive controlled inputs from a background sandbox — so when in
   doubt, run in the foreground.
5. **Re-record after UI changes.** Selectors are coupled to the DOM; a redesign
   will break them. Re-run the affected script and spot-check a frame
   (`ffmpeg -ss <t> -i out.mp4 -frames:v 1 frame.jpg`).

## Lessons that save hours

- **Cache narration by index.** Each line is `l<i>.wav`; reuse it if present so
  re-records don't re-spend credits. To change a line, edit `LINES[i]` and delete
  that one `l<i>.wav`. **If you insert a line mid-list, shift the cached wavs up by
  one** (`mv l7.wav l8.wav; mv l6.wav l7.wav; …`) so only the new line synthesizes
  — the loop caches by index and does not check the text.
- **Quota resilience.** If any TTS call fails, fall back to **footage + music
  only** and estimate each line's duration from word count, so the build always
  produces an mp4.
- **Trim the load-in lead.** Record the goto/seed/hydrate, mark `contentStart` once
  the first shot is settled, and `-ss contentStart` it off the front so the video
  opens on a calm frame. Narration cues are shifted by `contentStart` at mux time.
- **Hold = max(narration, pan).** For active holds, pad to at least the line length
  so the voice is never cut off; if the interaction runs longer, let it.
- **Record sharp.** Use a large viewport (e.g. 1920×1200) and apply
  `document.documentElement.style.zoom` (e.g. 1.5) via an init script, so logical
  layout math stays at your design size but the frame renders crisper.
- **Kill choppy scrolling.** Fixed, full-viewport overlays with `mix-blend-*`
  (film grain, gradients) force a full recomposite every scroll frame. Hide them
  while recording: `page.addStyleTag({content: ".grain{display:none!important}"})`.
  Also hide any always-on CTA/footer you don't want in frame.
- **Drive controlled inputs robustly.** A React-controlled `<select>` ignores a
  plain `selectOption`. Set it with the native setter + a bubbling `change` event,
  and **retry until a confirmation locator appears**:
  ```js
  const setter = Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, "value").set;
  setter.call(el, value); el.dispatchEvent(new Event("change", { bubbles: true }));
  ```
- **Match typographic quotes in selectors.** If headings render curly quotes
  (`'`, `'`, `"`), a regex with a straight `'` won't match. Use `.` for the
  apostrophe: `/Tell us who you.re looking for/i`.
- **Locate questions/fields by their label, then act inside the block.** Robust to
  layout changes: `page.locator("label", { hasText: q }).first().locator("xpath=..")`
  then `.getByRole("radio"/"checkbox", { name })` within it.
- **Music bed + ducking.** Synthesize a looping ambient pad from chord sine waves
  with ffmpeg `aevalsrc`, then `sidechaincompress` the music under the narration so
  the voice always reads clearly. Keep music ~0.4 and let the compressor dip it.

## Template (`demo/build-<name>.mjs`)

```js
import { chromium } from "playwright";
import fs from "node:fs";
import path from "node:path";
import { execFileSync } from "node:child_process";

const DIR = path.resolve("demo");
const WORK = path.join(DIR, "name-work");
fs.mkdirSync(WORK, { recursive: true });
const KEY = fs.readFileSync(path.join(DIR, ".eleven.key"), "utf8").trim();
const BASE = process.env.BASE_URL || "http://localhost:3000";
const OUT = path.join(DIR, "name.mp4");
const W = 1920, H = 1200, ZOOM = 1.5;

const sh = (c, a) => execFileSync(c, a, { stdio: ["ignore", "pipe", "pipe"] }).toString();
const probe = (f) => parseFloat(sh("ffprobe", ["-v","error","-show_entries","format=duration","-of","default=noprint_wrappers=1:nokey=1", f]).trim());
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

// 1 line per scene, in scene order.
const LINES = [
  "Opening line — the hook, spoken over the first settled frame.",
  "Second line — narrate while the camera pans through the next section.",
  "Closing line — a calm sign-off.",
];
const TAILS = LINES.map(() => 1.4); // breathing room after each line

// ── ElevenLabs ──────────────────────────────────────────────────────────────
async function voiceId() {
  const r = await fetch("https://api.elevenlabs.io/v1/voices", { headers: { "xi-api-key": KEY } });
  const { voices } = await r.json();
  return (voices.find((v) => v.name.toLowerCase().startsWith("jessica")) || voices[0]).voice_id;
}
async function tts(id, text, out) {
  const r = await fetch(`https://api.elevenlabs.io/v1/text-to-speech/${id}?output_format=mp3_44100_128`, {
    method: "POST", headers: { "xi-api-key": KEY, "content-type": "application/json" },
    body: JSON.stringify({ text, model_id: "eleven_multilingual_v2",
      voice_settings: { stability: 0.55, similarity_boost: 0.8, style: 0, use_speaker_boost: true } }),
  });
  if (!r.ok) throw new Error(`tts ${r.status} ${await r.text()}`);
  fs.writeFileSync(out, Buffer.from(await r.arrayBuffer()));
}

// ── Ambient music bed (Am–F–C–G pad) ────────────────────────────────────────
function buildMusic(total, out) {
  const chords = [[220,261.63,329.63],[174.61,220,261.63],[261.63,329.63,392],[196,246.94,293.66]];
  const CH = 6;
  const files = chords.map((freqs, i) => {
    const expr = freqs.map((f) => `0.17*sin(2*PI*${f}*t)`).join("+");
    const f = path.join(WORK, `chord${i}.wav`);
    sh("ffmpeg", ["-y","-f","lavfi","-i",`aevalsrc=${expr}:duration=${CH}:sample_rate=44100`,
      "-af", `afade=t=in:d=1.4,afade=t=out:st=${CH-1.4}:d=1.4`, f]);
    return f;
  });
  const list = path.join(WORK, "chords.txt");
  fs.writeFileSync(list, files.map((f) => `file '${f}'`).join("\n"));
  const loop = path.join(WORK, "loop.wav");
  sh("ffmpeg", ["-y","-f","concat","-safe","0","-i",list,"-c","copy", loop]);
  const loops = Math.ceil(total / (chords.length * CH)) + 1;
  sh("ffmpeg", ["-y","-stream_loop",String(loops),"-i",loop,"-af",
    ["aecho=0.8:0.88:70|130:0.25|0.18","lowpass=f=2200","tremolo=f=0.12:d=0.25",
     "aformat=channel_layouts=stereo","afade=t=in:d=0.5",
     `afade=t=out:st=${Math.max(0,total-3)}:d=3`,`atrim=0:${total}`].join(","), out]);
}

// ── Choreography helpers ────────────────────────────────────────────────────
async function smoothTo(page, y, ms) {
  await page.evaluate(({ y, ms }) => new Promise((res) => {
    const s = window.scrollY, d = y - s, t0 = performance.now();
    const e = (t) => (t < 0.5 ? 2*t*t : -1 + (4-2*t)*t);
    const f = (n) => { const p = Math.min(1, (n-t0)/ms); window.scrollTo(0, s + d*e(p)); p < 1 ? requestAnimationFrame(f) : res(); };
    requestAnimationFrame(f);
  }), { y, ms });
}
async function scrollToText(page, text, ms = 1500, offset = 150) {
  const y = await page.evaluate(({ text, offset }) => {
    const all = [...document.querySelectorAll("h1,h2,h3,h4,p,span,button,a,li")].filter((e) => e.getClientRects().length);
    let el = all.find((e) => (e.textContent || "").trim() === text)
          || all.filter((e) => (e.textContent || "").includes(text)).sort((a,b)=>a.textContent.length-b.textContent.length)[0];
    return el ? el.getBoundingClientRect().top + window.scrollY - offset : null;
  }, { text, offset });
  if (y != null) await smoothTo(page, Math.max(0, y), ms);
}

// ── 1) synthesize narration (cached), measure durations ─────────────────────
const clips = []; let vid = null, narrationOk = true;
for (let i = 0; i < LINES.length; i++) {
  const mp3 = path.join(WORK, `l${i}.mp3`), wav = path.join(WORK, `l${i}.wav`);
  if (fs.existsSync(wav)) { clips.push({ wav, dur: probe(wav) }); continue; }
  try {
    if (!vid) vid = await voiceId();
    await tts(vid, LINES[i], mp3);
    sh("ffmpeg", ["-y","-i",mp3,"-ar","44100","-ac","2", wav]);
    clips.push({ wav, dur: probe(wav) });
  } catch (e) {
    narrationOk = false;
    clips.push({ wav: null, dur: LINES[i].trim().split(/\s+/).length / 2.3 });
  }
}

// ── 2) record, holding each shot for its line ───────────────────────────────
const browser = await chromium.launch({ channel: "chrome", headless: true });
const context = await browser.newContext({ viewport: { width: W, height: H }, recordVideo: { dir: WORK, size: { width: W, height: H } } });
await context.addInitScript((z) => {
  const apply = () => { try { document.documentElement.style.zoom = String(z); } catch {} };
  apply(); document.addEventListener("DOMContentLoaded", apply);
}, ZOOM);
const page = await context.newPage();
page.setDefaultTimeout(20000);
const video = page.video();
const t0 = Date.now();

await page.goto(`${BASE}/`, { waitUntil: "domcontentloaded" });
await page.evaluate(() => (document.fonts ? document.fonts.ready : null));
await page.addStyleTag({ content: ".grain{display:none !important}" }); // hide blend overlays
await sleep(1500);
const contentStart = (Date.now() - t0) / 1000; // trim everything before this

const actions = [
  async () => { await sleep(300); },                                  // still hook
  { action: async () => { await scrollToText(page, "Some Heading"); await sleep(300); },
    hold:   async () => { await smoothTo(page, 1600, Math.max(4000, clips[1].dur*1000)); } }, // pan during line
  async () => { await smoothTo(page, 0, 2500); await sleep(500); },   // sign-off
];

const cues = [];
for (let i = 0; i < actions.length; i++) {
  const sc = actions[i];
  const act = typeof sc === "function" ? sc : sc.action;
  const hold = typeof sc === "function" ? null : sc.hold;
  await act();
  cues.push((Date.now() - t0) / 1000);
  if (hold) {
    const start = Date.now();
    await hold();
    const min = clips[i].dur + TAILS[i], elapsed = (Date.now() - start) / 1000;
    if (elapsed < min) await sleep((min - elapsed) * 1000);
  } else {
    await sleep((clips[i].dur + TAILS[i]) * 1000);
  }
}
await sleep(400);
await context.close();
await browser.close();

// ── 3) mux: trim lead, place narration at cues, duck music ──────────────────
const raw = await video.path();
const outDur = probe(raw) - contentStart;
const music = path.join(WORK, "music.wav");
buildMusic(outDur, music);

if (!narrationOk) {
  sh("ffmpeg", ["-y","-ss",String(contentStart),"-i",raw,"-i",music,
    "-filter_complex","[1:a]volume=0.7[a]","-map","0:v","-map","[a]",
    "-c:v","libx264","-preset","medium","-crf","18","-pix_fmt","yuv420p",
    "-c:a","aac","-b:a","192k","-movflags","+faststart","-t",String(outDur), OUT]);
} else {
  const lead = 0.35, narr = path.join(WORK, "narration.wav");
  const inputs = clips.flatMap((c) => ["-i", c.wav]);
  const fc =
    clips.map((c, i) => { const at = Math.max(0, Math.round((cues[i]-contentStart+lead)*1000)); return `[${i}:a]adelay=${at}|${at}[d${i}]`; }).join(";") +
    ";" + clips.map((_, i) => `[d${i}]`).join("") +
    `amix=inputs=${clips.length}:normalize=0:dropout_transition=0[mix];[mix]apad=whole_dur=${outDur},volume=1.25[a]`;
  sh("ffmpeg", ["-y", ...inputs, "-filter_complex", fc, "-map","[a]", narr]);
  sh("ffmpeg", ["-y","-ss",String(contentStart),"-i",raw,"-i",music,"-i",narr,
    "-filter_complex",[
      "[1:a]volume=0.42[m]","[2:a]asplit=2[k][n]",
      "[m][k]sidechaincompress=threshold=0.035:ratio=9:attack=15:release=350[md]",
      "[md][n]amix=inputs=2:normalize=0:duration=longest[a]",
    ].join(";"),
    "-map","0:v","-map","[a]",
    "-c:v","libx264","-preset","medium","-crf","18","-pix_fmt","yuv420p",
    "-c:a","aac","-b:a","192k","-movflags","+faststart","-t",String(outDur), OUT]);
}
console.log(`DONE: ${OUT}  (${outDur.toFixed(1)}s)`);
```

## Adapting it

- **New voice:** change the `voiceId()` name filter, or hardcode a `voice_id`.
- **Interactive tours:** add helpers like `answerChoice`/`clickTab` that scope to a
  block by label, retrying until a confirmation locator is visible. Put the
  clicking inside a scene's `hold` so it happens *during* the narration.
- **Verify before celebrating:** after a build, extract a frame at a scene's
  approximate timestamp and look at it; re-record if a selector silently no-op'd.
- **Distribution:** keep a small idempotent uploader (Drive API, S3, etc.) that
  updates each video *in place* so shared links survive re-records.
