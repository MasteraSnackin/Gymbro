# Gym Bro

An AI spotter for reviewing recorded workout sets.

## Description

Gym Bro helps lifters review their technique from an uploaded video. This prototype finds moments to review, measures joint angles with MediaPipe and uses those measurements to suggest coaching cues and an illustrative image correction.

This repository is a fork of [Aerovity/Gymbro](https://github.com/Aerovity/Gymbro), built for the VAST Builders Challenge, "Real-Time Video Agents Hack - LDN", on 17 October 2026. It analyses recorded clips, with a [cached demo](#try-the-cached-demo) available without API keys.

## Table of contents

- [Description](#description)
- [Features](#features)
- [Tech stack](#tech-stack)
- [Architecture Overview](#architecture-overview)
- [Installation](#installation)
- [Usage](#usage)
- [Configuration](#configuration)
- [Screenshots and demo](#screenshots-and-demo)
- [API and CLI reference](#api-and-cli-reference)
- [Tests](#tests)
- [Limitations](#limitations)
- [Contributing](#contributing)
- [Licence](#licence)
- [Contact and support](#contact-and-support)

## Features

- Supports pull-ups, back squats, bench presses, deadlifts, overhead presses and biceps curls.
- Measures pose locally and presents angle-based feedback, a form score and coaching cues.
- Adds streaming AI coaching, optional speech and image corrections when providers are configured.
- Compares before/after images with skeleton overlays, flagged moments, provider traces and PNG export.
- Shows interactive 3D pose replay with rule-derived target skeletons in split or ghost view.
- Provides a cached pull-up demo and local fallbacks when AI services are unavailable.

## Tech stack

| Area | Technologies |
| --- | --- |
| App and server | Next.js App Router, React, TypeScript, Node.js route handlers |
| Interface | Tailwind CSS, Base UI, shadcn, Motion, Zustand |
| Pose and 3D | MediaPipe Tasks Vision, Three.js, React Three Fiber, Drei |
| AI integrations | Cosmos-compatible endpoint, NVIDIA Nemotron Omni, OpenAI Responses and Images APIs |
| Validation and testing | Zod, Vitest, Testing Library, Playwright, ESLint |

The repository pins dependency versions in [package.json](package.json) and [package-lock.json](package-lock.json).

## Architecture Overview

```mermaid
flowchart LR
  User[User] --> Browser[Browser app]
  Browser -->|Local video frames| Pose[MediaPipe pose and exercise rules]
  Browser -->|Sampled images and measurements| API[Next.js API routes]
  API -->|Scan fallback chain| Scan[Cosmos / Nemotron / GPT vision]
  API -->|Coach and image edit| OpenAI[OpenAI]
  API -->|Reports and edited images| Browser
  Browser -->|Measure edited image| Pose
  Fixtures[Committed demo assets] --> Browser
```

The browser samples timestamped frames and measures pose locally. Next.js routes try Cosmos, Nemotron Omni and GPT vision in order for scanning, then request coaching and an image edit; the browser re-measures the edit against the primary target. Results live in browser memory, and committed files supply the demo; the app has no database or saved session history.

Read [ARCHITECTURE.md](ARCHITECTURE.md) for component responsibilities, data flows, deployment details and design trade-offs.

Download the [interactive architecture map](docs/diagrams/gymbro.architecture.html) and open it in a browser to search components, trace paths and explore guided views. GitHub displays the HTML source rather than running the viewer inline. The [editable map](docs/diagrams/gymbro.architecture.json) and [delivery receipt](docs/diagrams/gymbro.architecture.receipt.json) are saved alongside it.

In the standard upload flow, the original video stays in the browser. Live analysis sends sampled JPEGs or a contact sheet, a cropped still and pose measurements to the server and relevant AI providers; their retention policies apply. MediaPipe can fetch model and WASM files from Google and jsDelivr if local assets fail to load.

## Installation

- Git and npm, with Node 22.22.2+ within 22.x, 24.15.0+ within 24.x, or 26+, to satisfy the locked development dependencies.
- Google Chrome for browser tests.
- FFmpeg for phone-video conversion and the upload and error browser tests.

```bash
git clone https://github.com/MasteraSnackin/Gymbro.git
cd Gymbro
npm ci
cp .env.example .env.local
npm run dev
```

Open [localhost:3000](http://localhost:3000). You can leave the API keys blank to try the demo or local upload analysis.

Installation copies MediaPipe WASM into `public/mediapipe/wasm` and downloads missing heavy and lite pose models into `public/models`. These assets are also committed. Downloads can fail with a warning, so check the local assets before relying on pose analysis offline.

## Usage

### Try the cached demo

Choose the sample clip on the landing page, press `D`, or open [the pull-up demo](http://localhost:3000/analyze?demo=pullup). This uses committed fixtures and needs no API keys.

### Analyse your own clip

1. Choose the exercise before uploading. Explicit selection is particularly useful without a remote scan provider: automatic selection falls back to pull-up in that case.
2. Upload an MP4, MOV, M4V or WebM file that your browser can decode, with the relevant joints visible. Clips must be no longer than 60 seconds and no larger than 200 MiB. Selecting a file starts analysis.
3. Review the flagged moment, measured angles and coaching cues. Use `S` to toggle the skeleton and angles, or `3` to toggle the 3D view.
4. Compare the suggested correction and inspect the provider trace to see which services or fallbacks ran.

| Exercise | Suggested camera position |
| --- | --- |
| Pull-up | Directly in front or behind, with the bar and whole body visible |
| Back squat | Side view at hip height, with the whole body visible |
| Bench press | Foot end at about 45 degrees, with the bar and both elbows visible |
| Deadlift | Side view, with the bar and whole body visible |
| Overhead press | Side view, with the bar and whole body visible |
| Biceps curl | Side view, with the upper body and elbows visible |

When AI services fail or are unconfigured, uploads fall back to local pose analysis, rule-based coaching and a target ghost skeleton. The ghost does not go through generated-image verification.

For an iPhone HEVC clip that Chrome cannot decode, try Safari or convert it with FFmpeg:

```bash
npm run prep-video -- IMG_0618.MOV out.mp4 4 15
```

This starts at 4 seconds and exports a 15-second H.264 clip without audio. The command overwrites an existing output file.

### Run a production build locally

```bash
npm run build
npm run start
```

## Configuration

Edit `.env.local`, using [.env.example](.env.example) as the reference. Keep credentials on the server and out of commits. Restart the server after changing them.

| Variable | Purpose |
| --- | --- |
| `COSMOS_BASE_URL`, `COSMOS_API_KEY` | Primary scan endpoint and its bearer token. The token can be blank for a local service without authentication. |
| `NVIDIA_API_KEY` | Enables the hosted Nemotron Omni scan fallback. |
| `OPENAI_API_KEY` | Enables GPT vision fallback, streaming coaching and corrected-image generation. |
| `DEMO_MODE=1` | Substitutes cached pull-up responses in the AI analysis routes. |
| `MOCK_AI=1` | Uses the same route fixtures; used by browser tests. |

<details>
<summary>Optional model and provider settings</summary>

| Variable | Purpose |
| --- | --- |
| `COSMOS_MODEL` | Blank discovers the model through `/v1/models`. |
| `COSMOS_INPUT_MODE` | `video_frames` by default; also accepts `video_url` or `image_frames`. Rejected video-frame requests can retry with image frames. |
| `NVIDIA_BASE_URL`, `NEMOTRON_MODEL` | Override the hosted endpoint or model. |
| `OPENAI_COACH_MODEL`, `OPENAI_PICK_MODEL` | Repository default: `gpt-6.1-sol`. |
| `OPENAI_IMAGE_MODEL`, `OPENAI_IMAGE_QUALITY` | Repository defaults: `gpt-image-2.5-flare` and `medium`. |
| `OPENAI_IMAGE_INPUT_FIDELITY` | Leave blank unless the image model supports the setting. |
| `OPENAI_TTS_MODEL`, `OPENAI_TTS_VOICE` | Optional server speech. A blank model uses browser speech synthesis. |

[.env.example](.env.example) also lists frame-rate and timeout settings. Model names are configuration defaults; access depends on your provider account.

</details>

Use the sample demo URL for cached playback. Keep both flags at `0` when analysing your own clip: mocked routes return pull-up fixtures even for other uploads.

`GET /api/health` reports mode and configuration flags. `GET /api/health?probe=1` also probes the configured Cosmos model list and OpenAI coach model. It does not test Nemotron, image editing, speech or a complete analysis run.

## Screenshots and demo

These screens show a synthetic cached run built around a real pull-up clip. The [demo metadata](public/demo/pullup/meta.json) records real MediaPipe output for the before frame, while the after image is a later video frame used as a placeholder. Demo playback does not show live provider calls or an AI-generated correction.

![Animated tour of cached results, coaching and 3D pose comparison](docs/screens/gymbro-demo.gif)

This silent loop uses fresh screenshots of the app's cached demo. It shows the photo results, split 3D comparison and ghost overlay alongside the coaching panel. The 3D target is rule-derived.

<details>
<summary>Static screenshots</summary>

![Cached pull-up results with measured angles and coaching](docs/screens/results-desktop.png)

![3D pose comparison in split view](docs/screens/3d-replay-split.png)

</details>

The repository does not list a public deployment URL.

## API and CLI reference

The web app uses these Next.js routes. Request schemas are in [src/lib/schemas.ts](src/lib/schemas.ts), with browser helpers in [src/lib/api/client.ts](src/lib/api/client.ts).

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/api/health` | Mode and provider configuration; optional `?probe=1` reachability checks |
| `POST` | `/api/scan` | Timestamped frames to scan events and provider trace |
| `POST` | `/api/pick-frame` | GPT vision analysis of a contact sheet and timestamps |
| `POST` | `/api/coach` | Measurements and a still image to an SSE coaching stream (`delta`, `done`, `error`) |
| `POST` | `/api/correct` | Original still and target pose to an edited image |
| `POST` | `/api/tts` | Coaching text to MP3; returns `501` when server speech is unavailable |

Check a local server:

```bash
curl http://localhost:3000/api/health
curl 'http://localhost:3000/api/health?probe=1'
```

POST routes accept JSON and enforce a 4.4 MiB body cap. Validation errors return an `error` object with `code` and `message`; scan failures can also include a provider trace. The video conversion command in [Usage](#usage) is a development utility; the project has no separate analysis CLI.

## Tests

```bash
npm run typecheck
npm run lint
npm test
npm run test:cov
npm run test:e2e
```

Vitest covers unit and route integration tests. Use `npm run test:watch` while developing. Playwright builds and starts a production server on port 3210, then runs desktop Chrome and selected mobile-emulation tests with mocked AI routes. Install Google Chrome first; upload and error suites skip when FFmpeg is missing.

Mocked tests check application behaviour without proving that live model credentials, provider access or generated corrections work.

## Limitations

- Pose measurements depend on camera angle, joint visibility and the selected movement phase. Several exercise targets are coaching heuristics, and some measurements use proxies such as wrist position for bar path. The app does not provide clinical measurements.
- The displayed form score combines angle-based penalties with a coach adjustment when a remote report is available.
- Generated-image verification checks one primary metric for reliability and a tolerance of 10 degree-equivalents. A failed first check can trigger one regeneration if the initial edit took less than 45 seconds. Passing this check does not establish correct whole-body form or preserved identity.
- The 3D target is a rule-derived visualisation, not a recording of the user performing corrected movement.
- `package.json` defines `record-demo`, but its target, `scripts/record-demo.ts`, is missing. The command cannot currently refresh the bundled fixtures.

## Contributing

Submit a focused [pull request to this fork](https://github.com/MasteraSnackin/Gymbro/pulls), with reproduction steps for bugs and screenshots for interface changes. Run the relevant checks above and describe whether you tested with fixtures, local fallbacks or live providers. Issues are disabled on this fork; report bugs shared with the original project through [upstream issues](https://github.com/Aerovity/Gymbro/issues).

[docs/BUILD.md](docs/BUILD.md) explains the implementation layout, and [docs/SPEC.md](docs/SPEC.md) records the design specification. Check the source for current behaviour where those documents describe planned work.

## Licence

The repository has no licence file or licence declaration in `package.json`. Ask the relevant rights holders about permission to reuse or distribute the code.

## Contact and support

- [This fork](https://github.com/MasteraSnackin/Gymbro), maintained under [MasteraSnackin](https://github.com/MasteraSnackin).
- [Original project](https://github.com/Aerovity/Gymbro), by [Aerovity](https://github.com/Aerovity).
- [Upstream issues](https://github.com/Aerovity/Gymbro/issues) for bugs shared with the original project.
