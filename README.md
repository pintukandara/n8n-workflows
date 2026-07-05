# Instagram Reels Automation (with Overlay)

An end-to-end **n8n workflow** that turns Hindi newspaper e-paper images into AI-generated,
anchor-style news Reels — automatically summarized, scripted, fact-checked, rendered with
LTX-Video via ComfyUI, overlaid with a ticker/branding using FFmpeg, and published straight
to **Instagram** (and Facebook) via the Graph API, with a **Telegram human-approval gate**
before publishing.

> Repo: https://github.com/pintukandara/n8n-workflows

---

## What this workflow does (pipeline overview)

1. **Upload** – A form trigger (`On form submission`) accepts up to 4 newspaper page images.
2. **Extract & structure news** – Google Gemini (`Analyze an image`) reads the Hindi e-paper
   images and returns structured JSON: article titles, Hindi/English summaries.
3. **Clean & loop** – A Code node parses/cleans the Gemini JSON, then `Loop Over Items`
   processes each article one at a time.
4. **Reframe into a video script** – Gemini (`Message a model`) rewrites each summary into a
   punchy voiceover script (Hindi + English) and runs a **fact-deviation check** (flags any
   facts added/removed vs. the original article, with a deviation %).
5. **Human approval (Telegram)** – The reframed script, deviation %, and fact diffs are sent
   to Telegram (`Send message and wait for response`) with **Approve/Reject** buttons. Only
   approved articles continue.
6. **Generate the video prompt** – An AI Agent (`AI Agent1`, backed by Gemini) converts the
   approved script into a detailed **LTX-Video prompt**, keeping a fixed AI news-anchor
   persona/set identical across every video (only the spoken line & tone change).
7. **Render video (ComfyUI)** – The prompt is sent to a self-hosted **ComfyUI** instance
   (`ComfyUI1` node) running **LTX-Video 2.3 (22B, GGUF Q4_K_M)** to generate the anchor video.
8. **A second Telegram approval gate** reviews the rendered video before it goes further.
9. **Overlay branding (FFmpeg)** – The raw video is written to disk, and an FFmpeg command is
   built and executed (`Build FFmpeg Overlay Command` → `Run FFmpeg Overlay`) to burn in a
   location header and a scrolling news ticker with the article title.
10. **Upload & publish** – The final video is:
    - Uploaded to **Cloudinary** (`Upload an asset from file data`) to get a public video URL.
    - Published as an Instagram Reel via the **Facebook Graph API** nodes (create media
      container → poll/wait → publish).
11. **Error handling** – Any failed step (image analysis, article processing, upload) routes
    to dedicated Telegram notification nodes (`Notify - Image Analysis Error`,
    `Notify - Article Pipeline Error`) so failures never fail silently, and the loop
    continues with the next item.

---

## Requirements

### 1. n8n
- A running n8n instance (self-hosted recommended — this workflow uses `executeCommand`
  and local file read/write, which require **Docker/self-hosted n8n**, not n8n Cloud).
- Recommended: n8n running inside **Docker (WSL2 on Windows, or native Linux)** with FFmpeg
  installed in the same container/host so `n8n-nodes-base.executeCommand` can call `ffmpeg`.
- Node/community packages used:
  - `@n8n/n8n-nodes-langchain` (for the Gemini chat/agent nodes — `AI Agent1`,
    `googleGemini`, `lmChatGoogleGemini`)
  - A **ComfyUI** community/custom node (`ComfyUI1`) — install the appropriate ComfyUI n8n
    node/integration for your instance.

### 2. ComfyUI (self-hosted, for video generation)
- ComfyUI installed and reachable from n8n (same machine or via a tunnel, e.g. Cloudflare
  Tunnel, if n8n and ComfyUI are on different hosts/networks).
- **LTX-Video 2.3 (22B, GGUF Q4_K_M)** model loaded (or your preferred LTX-Video checkpoint).
- A GPU with sufficient VRAM for the model (this pipeline was built/tested on an RTX 5070,
  12GB VRAM). Adjust quantization/model size to fit your GPU.
- Note: on Blackwell/sm_120-class GPUs, SageAttention may be incompatible — use standard
  attention if you hit errors there.

### 3. FFmpeg
- FFmpeg installed on the machine/container that runs n8n's `executeCommand` node.
- A font file available for the ticker/header overlay text — the workflow expects:
  ```
  /usr/share/fonts/dejavu/DejaVuSans-Bold.ttf
  ```
  Install `fonts-dejavu-core` (or update the `FONT_PATH` in the "Build FFmpeg Overlay
  Command" node to point at any TTF font you have available).
- A writable local directory for temp video files — the workflow writes/reads from:
  ```
  /media/reel-processing/
  ```
  Make sure this path exists and is writable inside your n8n/Docker container (mount a
  volume if needed).

### 4. Credentials to configure in n8n
Set these up under **n8n → Credentials** before importing the workflow, then re-map each
node to your own credential (the JSON references credential IDs from the original author's
instance, which won't exist in yours):

| Credential type       | Used by nodes                                                   | Needed for |
|------------------------|------------------------------------------------------------------|------------|
| Google Gemini (PaLM) API | `Analyze an image`, `Message a model`, `Google Gemini Chat Model1` | Reading newspaper images, script reframing, prompt-generation agent |
| Telegram API           | `Send message and wait for response(1)`, `Notify - Image Analysis Error`, `Notify - Article Pipeline Error` | Human approval gates + error alerts |
| ComfyUI                | `ComfyUI1`                                                        | Sending the prompt & receiving the rendered video |
| Cloudinary API         | `Upload an asset from file data`                                  | Hosting the final video and getting a public URL |
| Facebook Graph API     | `Facebook Graph API2`, `Facebook Graph API3`                     | Creating the media container & publishing to Instagram (requires a Facebook Page linked to an Instagram Business/Creator account, and an app with `instagram_content_publish` / relevant Graph API permissions) |

### 5. Environment variables (`.env`)
Secrets (Gemini API key, Telegram bot token, Cloudinary key/secret, Facebook Graph API
token, ComfyUI auth) live in **n8n's own encrypted Credentials store** — they are never
written into the workflow JSON, so there's nothing to move for those.

The two non-secret but instance-specific IDs that *were* hard-coded in the JSON have been
parameterized to read from environment variables instead:

- `TELEGRAM_CHAT_ID` — used by all Telegram nodes (approval prompts + error alerts)
- `IG_BUSINESS_ACCOUNT_ID` — used by the Facebook Graph API nodes (`node` parameter)

Copy `.env.example` to `.env`, fill in your values, and load it into your n8n
container/process (e.g. via `docker-compose --env-file .env` or your compose file's
`env_file:` directive). You also need to set:

```
N8N_BLOCK_ENV_ACCESS_IN_NODE=false
```

n8n blocks node expressions from reading `$env` by default as a security measure (so that
anyone with workflow-edit access can't read your host's environment variables). Only set
this to `false` on an instance where you trust everyone who can edit workflows — for a
personal/self-hosted setup like this, that's usually fine.

### 6. Other things you'll need to update after importing
- **Credential IDs/names** — n8n will show these as broken until you re-select your own
  credentials in each node listed above.
- **ComfyUI prompt/workflow JSON** inside the `ComfyUI1` node — confirm it matches your
  ComfyUI workflow (model name, node IDs) if you're not using an identical ComfyUI setup.
- **Local file paths** (`/media/reel-processing/...`, font path) — adjust to match your
  container/host filesystem.

---

## Setup steps

1. Clone/download this repo and import `Instagram_Reels_Automation_with_overlay.json` into
   n8n (**Workflows → Import from File**).
2. Install any missing community nodes (`@n8n/n8n-nodes-langchain`, your ComfyUI node).
3. Create the credentials listed above in n8n and re-map each node to use them.
4. Make sure FFmpeg is installed and the font path + `/media/reel-processing/` directory
   exist and are writable in your n8n environment.
5. Confirm ComfyUI is running, reachable from n8n, and has the LTX-Video model loaded.
6. Copy `.env.example` to `.env`, fill in `TELEGRAM_CHAT_ID` and `IG_BUSINESS_ACCOUNT_ID`,
   set `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`, and load the file into your n8n
   container/process.
7. Restart n8n so it picks up the new environment variables.
8. Activate the workflow and test it by submitting 1 image through the form trigger first,
   approving each Telegram prompt, and confirming a Reel gets created end-to-end before
   scaling up to full batches.

---

## Notes / limitations

- This workflow assumes a **self-hosted / Docker** n8n instance due to `executeCommand`
  and local filesystem access — it will not run as-is on n8n Cloud.
- The two Telegram "wait for response" steps mean the workflow **pauses** until you approve
  or reject in Telegram — make sure your n8n instance is configured for long-running/waiting
  workflows (webhooks enabled, no aggressive execution timeouts).
- Costs to be aware of: Gemini API usage, ComfyUI/GPU compute, Cloudinary storage/bandwidth,
  and Meta Graph API rate limits.

---

## License

This project is licensed under the [MIT License](./LICENSE).