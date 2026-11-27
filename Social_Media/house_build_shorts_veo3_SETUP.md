# House Build Shorts with Veo 3 (YouTube, Facebook, Instagram, TikTok)

Workflow file: `house_build_shorts_veo3_multiplatform.json`

Every 12 hours this workflow picks the next **Pending** topic from a Google Sheet, writes an 8-scene script,
generates 8 vertical Google Veo 3 clips, stitches them into one 60-second 9:16 video, and posts it to
YouTube Shorts, a Facebook Page, Instagram Reels and TikTok. It follows the same pattern as
`Video_Automation__images_only_.json` (Sheet → LLM scenes → loop → Drive → Creatomate → publish).

## Why 8 clips?
Veo 3 makes clips of at most 8 seconds. The workflow generates 8 clips (one per build stage: plot, foundation,
walls, roof, windows/cladding, interior, landscaping, final reveal) and Creatomate plays them back to back,
trimming each to 7.5 s, so the result is exactly 60 s. Change `scene_count` / `clip_seconds` in **Config** to adjust.
Veo 3's own audio (construction sounds) is kept.

Clips are made from text only, so the script repeats the same house description and the same fixed camera angle
in every prompt to keep the house consistent across clips.

## Nodes used (all built into n8n)
| Step | Node |
|---|---|
| Topics | Google Sheets |
| Script | Basic LLM Chain + Google Gemini Chat Model + Structured Output Parser |
| Clips | Google Gemini → Video → Generate (model `models/veo-3.0-generate-001`) |
| Clip storage | Google Drive (public folder so Creatomate can read the clips) |
| Stitching | HTTP Request → Creatomate API (RenderScript) |
| YouTube | YouTube → Video → Upload |
| Facebook | Facebook Graph API → `{page-id}/videos` on graph-video.facebook.com |
| Instagram | Facebook Graph API → `{ig-user-id}/media` (REELS) then `media_publish` |
| TikTok | HTTP Request → TikTok Content Posting API (`/v2/post/publish/video/init/`, FILE_UPLOAD) |

n8n has no TikTok or Veo-specific node, so TikTok uses HTTP Request and Veo uses the official Google Gemini node.

## Setup
1. **Google Sheet** with columns `Topic | Status | Title | YouTube URL | Video URL`.
   Add rows like `Modern farmhouse built from scratch` with Status `Pending`.
   Select the sheet in **Get Pending Topic** and **Mark Row as Posted**.
2. **Config** node: fill in `fb_page_id`, `ig_user_id` (Instagram Business account linked to the Page),
   `tiktok_privacy_level`, `youtube_privacy`.
3. **Credentials** (create in n8n, never paste keys into nodes):
   - Google Gemini (PaLM) API: an API key from Google AI Studio with Veo access (paid tier).
   - Google Drive OAuth2 and Google Sheets OAuth2.
   - YouTube OAuth2.
   - Facebook Graph API: a long-lived **Page** access token with `pages_manage_posts`, `pages_read_engagement`,
     `publish_video`, `instagram_basic`, `instagram_content_publish`.
   - Header Auth for Creatomate: name `Authorization`, value `Bearer YOUR_CREATOMATE_KEY`.
   - OAuth2 API for TikTok: auth URL `https://www.tiktok.com/v2/auth/authorize/`, token URL
     `https://open.tiktokapis.com/v2/oauth/token/`, scope `video.publish`. TikTok uses `client_key`
     instead of `client_id`, so check the TikTok developer docs if the token exchange fails.
4. Run once manually, then activate.

## Things to know
- **TikTok:** until your TikTok app passes TikTok's audit, posts can only be private (`SELF_ONLY`).
  Read the allowed values from TikTok's `creator_info/query` endpoint before switching to `PUBLIC_TO_EVERYONE`.
- **Veo people setting:** the clip node uses `personGeneration = allow_all`. If Google returns a region error, set it to `allow_adult`.
- **Cost:** 8 Veo 3 clips per video. Check current Veo pricing before scheduling every 12 h.
- **Drive links:** clips are shared as "anyone with the link" so Creatomate can download them.
- The render and Instagram steps poll in a loop and stop with an error if the service reports a failure.
