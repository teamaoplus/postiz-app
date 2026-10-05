# Video clipping

Clipping turns a long YouTube video into short vertical clips (10–90 seconds) with burned-in captions. AI picks the best moments. Every clip is saved to the media library, and you can have a **draft** post created for each clip on the channels you choose. Nothing is scheduled or published automatically.

## How to use it

Postiz has no clipping screen. You start clips by talking to the AI agent:

- the **Agent** page in Postiz, or
- any AI app connected to Postiz over MCP (Claude, ChatGPT, Cursor, ...). See [Public API → MCP](./public-api.md#mcp).

### Steps

1. Ask the agent, for example: *"Clip https://www.youtube.com/watch?v=... into 5 clips and make drafts for my LinkedIn and TikTok channels."*
2. The agent asks how the horizontal video should fill the vertical clip. Answer with one of:
   - **blur**: keeps the whole picture, centered over a blurred copy of itself. Always safe.
   - **crop**: fills the frame with the middle of the picture and cuts the sides away. There's no face tracking, so a speaker who isn't in the center gets cut out.
3. The agent starts the clipping and gets back a clipping ID. Clipping takes several minutes.
4. Follow the progress:
   - MCP clients that support MCP Apps show a progress widget: **Analysing → Transcribing → Picking clips → Rendering → Done**. The widget reports the finished clips back into the conversation.
   - Everywhere else, including the Agent page, ask *"how is the clipping going?"*. The agent checks the status and lists the clips when they're ready.
5. Find the clips in **Media**. If you named channels, each clip also appears as a draft in the **Calendar**, placed in the next free time slots, with the clip attached and a suggested post text. Review the drafts, edit them, and schedule them.

### Options

| Option | Values | Default |
| --- | --- | --- |
| Video URL | A YouTube URL (other sites aren't supported) | required |
| Channels | Up to 20 channels to create drafts on | none: clips only go to the media library |
| Number of clips | 1–10 (a maximum: fewer are made if the video doesn't have enough good moments) | 5 |
| Fit | `blur` or `crop` | The agent always asks. The REST API defaults to `blur`. |

### What happens behind the scenes

1. **Analysing**: reads the video's metadata and captions. If there are no usable captions, it downloads the audio instead.
2. **Transcribing**: only when there were no captions. The audio is transcribed with Deepgram.
3. **Picking clips**: an OpenAI model reads the transcript and picks the moments, writing a title and a post text for each.
4. **Rendering**: each clip is fetched, cut to 1080×1920, captioned, and uploaded with a thumbnail.
5. **Drafts**: one draft per clip per selected channel. Disabled or deleted channels are skipped.

A clipping can finish as `completed` while some of its clips failed. Each clip has its own status and error.

## Billing and limits

Clipping uses **clipping minutes**: one minute per minute of source video, rounded up. They're charged once the real duration is known, before any rendering. If the clipping fails and produced no clip, the minutes are refunded.

| Plan | Clipping minutes per month |
| --- | --- |
| Free | 0 |
| Standard | 60 |
| Team | 120 |
| Pro | 300 |
| Ultimate | 600 |

Other limits:

- The video can't be longer than your remaining minutes, and never longer than 180 minutes.
- Only one clipping runs per organization at a time.
- At most 20 clippings can start per organization per day.
- Clipping isn't available during a trial.
- A clipping that hasn't finished after 4 hours is closed as failed and refunded.

Self-hosted installs without Stripe have no minute metering. The 180-minute cap still applies.

## Troubleshooting

| Message | Cause |
| --- | --- |
| `Clipping is not available` | The feature isn't configured on this server (see below), or Temporal is down |
| `Only YouTube videos can be clipped` | The URL isn't a YouTube URL |
| `Clipping is not available in trial mode` | The organization is still on a trial |
| `No clipping minutes are left on this account for this month` | The monthly minutes are used up |
| `A clipping is already running, wait for it to finish` | Another clipping is still in progress |
| `Too many clippings were started today, try again tomorrow` | The 20-per-day limit was hit |
| `This video is N minutes long, ...` | The video is longer than the remaining minutes or the 180-minute cap |
| `Channel X not found` | A channel ID doesn't belong to the organization |

## Enabling clipping (self-hosted)

Clipping is only enabled when **all** of these are set (`UploadFactory.clippingEnabled()`):

```bash
STORAGE_PROVIDER="cloudflare"           # clipping hands presigned URLs to the media service
RUNPOD_API_KEY=""                       # must have access to both endpoints below
RUNPOD_INGEST_ENDPOINT_ID=""            # CPU endpoint of postiz-uploader (ingest jobs)
RUNPOD_CLIPPER_ENDPOINT_ID=""           # GPU endpoint of postiz-uploader (clip jobs)
DEEPGRAM_API_KEY=""                     # transcribes videos without usable captions
OPENAI_API_KEY=""                       # picks the clips
```

The `CLOUDFLARE_*` storage variables must be configured too. The postiz-uploader endpoints need their own Oxylabs account to fetch from YouTube. The orchestrator must be running, because each clipping runs as a Temporal workflow (`clippingWorkflow`, ID `clipping_{id}`), which you can inspect in the Temporal UI.

When any variable is missing, the clipping MCP tools are hidden and the API answers `503 Clipping is not available`.

## REST endpoints (web app session)

Clipping isn't part of the public API (`/public/v1`). The web app's own backend exposes it under the logged-in user's session:

| Method | Path | Description |
| --- | --- | --- |
| POST | `/clipping` | Start a clipping: `{ "url", "integrations"?, "clips"?, "fit"? }` returns `{ "id" }` |
| GET | `/clipping?page=1` | List clippings, 20 per page: `{ pages, results }` |
| GET | `/clipping/:id` | One clipping with `status`, `error`, `title`, `duration` and its `clips` (`title`, `content`, `start`, `end`, `status`, `error`, `path`, `thumbnail`, `mediaId`) |

Clipping `status` moves through `analysing` → `transcribing` (only when needed) → `picking` → `rendering` → `completed` or `failed`.

## Code map

| Piece | Location |
| --- | --- |
| Service (all logic) | `libraries/nestjs-libraries/src/database/prisma/clipping/clipping.service.ts` |
| Repository | `libraries/nestjs-libraries/src/database/prisma/clipping/clipping.repository.ts` |
| DTO | `libraries/nestjs-libraries/src/dtos/clipping/clipping.dto.ts` |
| Controllers | `apps/backend/src/api/routes/clipping.controller.ts`, `clipping.widget.controller.ts` |
| Workflow and activities | `apps/orchestrator/src/workflows/clipping.workflow.ts`, `apps/orchestrator/src/activities/clipping.activity.ts` |
| AI tools | `libraries/nestjs-libraries/src/chat/tools/clipping.tool.ts`, `clipping.status.tool.ts`, `clipping.widget.ticket.tool.ts` |
| Progress widget | `libraries/nestjs-libraries/src/chat/ui/clipping.widget.ts` |
| Media service contract | `libraries/nestjs-libraries/src/upload/clipping.processor.interface.ts` |
| Plan minutes | `libraries/nestjs-libraries/src/database/prisma/subscriptions/pricing.ts` (`clipping_minutes`) |
