---
name: custom-playbook-260p255y
description: Publish a LinkedIn post together with an uploaded video on the user's
  behalf from a short description and a video file, capture the URL of the newly created
  post, then write a Slack message from a second description that includes that link
  and post it to a channel the user names. Use when someone wants one announcement
  published on LinkedIn and shared to a Slack channel.
---

LinkedIn Post + Slack Announcement

You are the agent that publishes one announcement on LinkedIn **as the connected user** and then shares the live post link in a Slack channel. The order matters: the Slack message needs the LinkedIn post URL, so LinkedIn is posted first and Slack second.

## Inputs

- **LinkedIn post description** (`linkedin_post_description`, required) — what the LinkedIn post should say: topic, key points, tone, any hashtags or mentions. If it is already the finished post text, post it as written (light formatting only).
- **Slack message description** (`slack_message_description`, required) — what the Slack message should say. The LinkedIn post link is always added to it.
- **Slack channel** (`slack_channel`, required) — the channel to post in, e.g. `#marketing` or `marketing` (a channel ID such as `C0123456789` is also accepted).

- **Video** (`video_file`, required) — a video file from the room's resources. The LinkedIn post is always published together with this video; a text-only post is never acceptable.

## Connectors (never ask for credentials)

Call `list_vault_apis` and use the entries that are `is_configured: true` with `oauth_flow_state: active`:

- **LinkedIn** — `api_origin` `api.linkedin.com`. Several LinkedIn entries may exist; use the configured one. Needs scope `w_member_social` (and `openid`/`profile` for the member id).
- **Slack** — `api_origin` `slack.com`.

Pass the chosen ids in `vault_api_ids` to `execute_python`; the platform injects the user's authorization header. If either connector is missing or not configured, tell the user to connect it and stop (`report_playbook_progress(status="failed")`). Never fabricate a post URL, and never ask the user to type a token or id.

## Clarifying questions

The user asked to be asked if anything is unclear. Settle that **before the first stage**, not partway through: if a required input is empty, or the channel name is ambiguous or cannot be understood, ask one concise question (`request_user_answers`) and wait. Once the stages start, do not stop to ask. Resolve small gaps with sensible defaults and state them in the final summary. In a scheduled or unattended run, skip questions and use defaults, or fail with a clear note if a required input is missing.

## Rules for this playbook

- **Post to LinkedIn exactly once.** Never retry the LinkedIn publish call after it returned success or after an ambiguous response; if unsure, first check for the post before retrying. A duplicate public post cannot be undone by this playbook.
- The post and the video go out together or not at all: never publish a text-only post, and never publish a post whose video upload failed or is missing. If the video cannot be attached for any reason, fail the run before posting anything.
- Uploading the video does not publish anything; only the single `POST /rest/posts` call in `post_linkedin` makes the post public. Never attach a video that did not reach status `AVAILABLE`.
- Post to Slack only after a real LinkedIn post URL exists.
- If the Slack post fails after LinkedIn succeeded, do not re-post to LinkedIn. Report the LinkedIn URL and the Slack error so the user can finish by hand.
- Do not invent facts, statistics, names or claims that are not in the descriptions. Keep the LinkedIn post to at most 3000 characters.
- Python code used for API calls writes diagnostics to stderr only; stdout carries the result table.

## Completeness contract

Before finishing, confirm each of these exists, and name any that does not and why:

- `content.drafts` — final LinkedIn post text and Slack message text (with a link placeholder).
- `slack.channel_id` — the verified Slack channel ID.
- `linkedin.video_urn` — the URN of the uploaded video, confirmed `AVAILABLE` on LinkedIn.
- `linkedin.post_url` — the URL of the newly published LinkedIn post, which includes the video.
- `slack.message_posted` — the Slack message (including the LinkedIn URL) was posted; report channel and message timestamp.

## Failure handling — alert #production-issues (only on failure)

If any stage fails and the run's deliverables are not completed (a connector is missing, the Slack channel cannot be resolved, the video upload or processing fails, LinkedIn rejects the post, the Slack post fails, or any unexpected error), post one alert to the Slack channel `#production-issues`. **When every required output was produced, never post to this channel.** A run that succeeds must not touch it.

Do this before reporting the stage `failed`, because `failed` ends the run:

1. Resolve `#production-issues` to its channel ID with `conversations.list` (same method as `verify_slack_channel`; match the name `production-issues` exactly). Use the Slack connector from `list_vault_apis`. Do not fall back to another channel. If the channel cannot be found, or Slack itself is the thing that failed and no alert can be posted, say so plainly in your final reply and still report the stage `failed`.
2. Post one message with `chat.postMessage` (JSON `channel`, `text`) in Slack mrkdwn, containing:
   - `:rotating_light: *LinkedIn Post + Slack Announcement* failed`
   - the *failed stage* (its plan id and label)
   - the *error message*: the exact error or API response summary. Include the HTTP status and Slack/LinkedIn error code. Truncate it to about 1,500 characters. Never include tokens, authorization headers, or other secrets.
   - *what was completed before the failure*, so nobody duplicates work: for example the LinkedIn post URL if the post was already published, the uploaded video URN, or the Slack channel that was resolved. Say explicitly if the LinkedIn post is already live.
   - the *link to the room and run*: the playbook page `https://app.corvic.live/room/<room_id>/playbook/custom-playbook-260p255y`, plus the run's chat thread id. Take the room id and thread id from the `current_url` in `get_environment` (`client_contextual_info`) when present. Otherwise take them from this thread's `sys/created-by/org:...:room:<room_id>:agent:<agent_id>:chatThread:<thread_id>` trait. Never invent an id.
3. Post the alert at most once per run. Do not retry it if it fails; report that in the final reply.
4. Then call `report_playbook_progress(step_id="<failed stage id>", status="failed", note="<short user-safe reason>")`.

A LinkedIn post that already went live is never re-posted as part of failure handling. If only the Slack announcement failed, the alert reports the LinkedIn URL so the announcement can be sent by hand.

## How this Playbook runs

Immediately before each stage, call `use_skill(skill_name="custom-playbook-260p255y", step_id="<id>")` to load that stage's instructions, using the plan ids: `draft_content`, `verify_slack_channel`, `upload_linkedin_video`, `post_linkedin`, `post_slack`. Loading a stage marks it started and loading the next marks the previous one complete, so do not emit `report_playbook_progress` for `started` or `done` on intermediate stages. Call `report_playbook_progress` only (a) with `status="failed"` and a short user-safe `note` when a stage cannot complete, which ends the run, and (b) with `status="done"` once for the final stage `post_slack`, after the Slack message has actually been posted. `draft_content`, `verify_slack_channel` and (after the channel check) `upload_linkedin_video` are independent of the drafting and may run in parallel.

Drive the run through every stage to the end of the contract in one go. Do not pause to ask whether to continue, request permission, or post progress check-ins; the only early exit is a hard, unrecoverable error, handled as described in **Failure handling** (alert `#production-issues`, then report `failed`). Everywhere a stage below says to fail or stop, that means: run the Failure handling steps first.

## Step: draft_content — Draft the LinkedIn post and the Slack message

Produces `content.drafts`. Runs in parallel with `verify_slack_channel`.

1. Read the two descriptions from the inputs.
2. Write the **LinkedIn post**: a strong first line, short paragraphs, plain text (LinkedIn does not render Markdown), 0–5 relevant hashtags at the end only if they fit the description. Keep the user's tone and facts; add nothing that was not given. Maximum 3000 characters.
3. Write the **Slack message**: concise, in the tone the description asks for, using Slack mrkdwn (`*bold*`, `<url|text>`). Place the literal token `{{LINKEDIN_URL}}` where the link belongs; if the description does not say where, put it on its own line at the end. The link must appear at least once.
4. Keep both texts in working memory (and optionally in a small table) for the next stages. Do not post anything in this stage.

## Step: verify_slack_channel — Verify the Slack channel and connectors

Produces `slack.channel_id`. Runs in parallel with `draft_content`.

1. `list_vault_apis`: find the configured LinkedIn and Slack connectors. If either is missing, call `report_playbook_progress(step_id="verify_slack_channel", status="failed", note="Connect <LinkedIn|Slack> and re-run")` and stop.
2. Resolve the Slack channel in `execute_python` with the Slack `vault_api_ids`, using `requests` with timeouts and `raise_for_status()`:
   - If the input looks like a channel ID (`^[CG][A-Z0-9]+$`), call `https://slack.com/api/conversations.info?channel=<id>`.
   - Otherwise strip a leading `#` and page through `https://slack.com/api/conversations.list?types=public_channel,private_channel&exclude_archived=true&limit=200` (follow `response_metadata.next_cursor`, at most 20 pages) to find an exact name match (case-insensitive).
   - Slack replies HTTP 200 even on errors, so always check the JSON `ok` field and report the `error` value.
3. If the channel cannot be found, say so and offer the closest names; in an interactive run ask the user once before any posting, otherwise fail with a clear note. Do not guess a different channel.
4. Output the channel ID and name. Write only to stderr; stdout carries the result table.

## Step: upload_linkedin_video — Upload the video to LinkedIn

Produces `linkedin.video_urn`. Depends on `verify_slack_channel`. The video is mandatory. If the `video_file` input is empty or the file cannot be found, fail the run with a clear note (`report_playbook_progress(status="failed")`); never continue to a text-only post. In an interactive run, if the video is missing, ask the user once for it before the stages start.

1. Locate the file among the room's resources (`fetch_feature_view` on the resource/source listing, matching the path or name). Stage it with `extract_tables` if needed so it is a file table (`{path, content, mime_type}`), and pass that table as `input_table_ids` to `execute_python` (read it from `$CORVIC_INPUT_TABLES`, then `requests.get(url)` + `pl.read_parquet`). The room's source listing only holds file metadata (`name`, `source_path`, `size`, `mime_type`), so run `extract_tables` on that resource first to get the file bytes as `{path, content, mime_type}`, then filter to the chosen file. Confirm it is a video (mime type `video/*`). LinkedIn's video API officially accepts MP4 only: if the file is not MP4 (for example `video/quicktime` / `.mov`, as screen recordings usually are), check whether `ffmpeg` is available in the sandbox (`subprocess`, `ffmpeg -version`) and, if so, convert it to H.264/AAC MP4 (`ffmpeg -i in.mov -c:v libx264 -preset veryfast -crf 23 -c:a aac -movflags +faststart out.mp4`, writing under `/tmp`, which counts against the 4 Gi budget) and upload the converted file, using its size for `fileSizeBytes`. If `ffmpeg` is not available, upload the original bytes as-is and rely on the `AVAILABLE` check below; if LinkedIn then reports `PROCESSING_FAILED`, fail the run with a note that the video needs to be re-uploaded as MP4. LinkedIn accepts video of about 3 seconds to 30 minutes and up to 5 GB; if it is clearly outside that or not a video, fail with a clear note rather than posting without it. Keep the whole file within the sandbox's 4 Gi memory budget.
2. Use `execute_python` with the LinkedIn `vault_api_ids`. Get the member id with `GET https://api.linkedin.com/v2/userinfo` (`sub`); owner is `urn:li:person:<sub>`. Use headers `LinkedIn-Version: 202609` (LinkedIn retires old versions; if the API answers 426 `NONEXISTENT_VERSION`, retry with the previous `YYYYMM`, e.g. 202608, 202607, …) and `X-Restli-Protocol-Version: 2.0.0`.
3. Initialize: `POST https://api.linkedin.com/rest/videos?action=initializeUpload` with body `{"initializeUploadRequest": {"owner": "urn:li:person:<sub>", "fileSizeBytes": <size>, "uploadCaptions": false, "uploadThumbnail": false}}`. The response `value` holds `video` (the video URN, `urn:li:video:...`), `uploadToken`, and `uploadInstructions` (a list of `{firstByte, lastByte, uploadUrl}` parts).
4. Upload each part: `PUT` the bytes `[firstByte, lastByte]` to its `uploadUrl` with `Content-Type: application/octet-stream` (do not add the LinkedIn auth header to the upload URL if the request fails with it; those URLs are pre-signed). Collect the `ETag` response header of every part, in order, using generous timeouts and up to 3 retries per part (retrying a part is safe).
5. Finalize: `POST https://api.linkedin.com/rest/videos?action=finalizeUpload` with `{"finalizeUploadRequest": {"video": "<video urn>", "uploadToken": "<uploadToken>", "uploadedPartIds": [<etags in order>]}}`.
6. Poll `GET https://api.linkedin.com/rest/videos/<url-encoded video urn>` every 10 seconds for at most 10 minutes until `status` is `AVAILABLE`. If it becomes `PROCESSING_FAILED` or never becomes available, fail the run with a clear note (nothing has been posted yet). Return the video URN as `linkedin.video_urn`.

## Step: post_linkedin — Publish the post on LinkedIn and capture its URL

Produces `linkedin.post_url`. Depends on `draft_content`, `verify_slack_channel` and `upload_linkedin_video` (so nothing is posted if the Slack target is invalid or the video upload failed).

Use `execute_python` with the LinkedIn `vault_api_ids`:

1. Get the member id: `GET https://api.linkedin.com/v2/userinfo`; the `sub` field is the member id. The author is `urn:li:person:<sub>`.
2. Publish with `POST https://api.linkedin.com/rest/posts` and headers `Content-Type: application/json`, `LinkedIn-Version: 202609` (LinkedIn retires old versions; if the API answers 426 `NONEXISTENT_VERSION`, nothing was posted, so retry with the previous `YYYYMM`, e.g. 202608, 202607, …), and `X-Restli-Protocol-Version: 2.0.0`. Body:
   ```json
   {
     "author": "urn:li:person:<sub>",
     "commentary": "<LinkedIn post text>",
     "visibility": "PUBLIC",
     "distribution": {"feedDistribution": "MAIN_FEED", "targetEntities": [], "thirdPartyDistributionChannels": []},
     "lifecycleState": "PUBLISHED",
     "isReshareDisabledByAuthor": false
   }
   ```
   Always include the video URN from `upload_linkedin_video` (never publish without it): add `"content": {"media": {"title": "<short title from the post's first line, max 100 chars>", "id": "<video urn>"}}` to the body. (The `ugcPosts` fallback uses `shareMediaCategory: VIDEO` with a `media` entry `{status: READY, media: <video urn>}` instead of `NONE`.)
   Escape reserved characters in `commentary` that LinkedIn treats specially (`( ) [ ] { } < > @ # * _ ~ \ |`) with a backslash, except for intentional hashtags, which must remain as plain `#tag`.
   If `/rest/posts` is unavailable for this app, fall back to `POST https://api.linkedin.com/v2/ugcPosts` with `specificContent.com.linkedin.ugc.ShareContent` (`shareCommentary.text`, `shareMediaCategory: NONE`) and `visibility.com.linkedin.ugc.MemberNetworkVisibility: PUBLIC`.
3. A successful create returns HTTP 201 with the post URN in the `x-restli-id` response header (for `ugcPosts` it is also the `id` in the body), e.g. `urn:li:share:123` or `urn:li:ugcPost:123`. Do **not** retry after a 201.
4. Build the URL: `https://www.linkedin.com/feed/update/<urn>/`. Print/return it as `linkedin.post_url`.
5. On HTTP 401/403 tell the user to reconnect LinkedIn (scope `w_member_social`) and fail the run. On 422/429/5xx report the exact error body (without secrets) and fail; do not post a second time without first checking that the first attempt did not publish.

## Step: post_slack — Add the link to the Slack message and post it

Produces `slack.message_posted`. Depends on `post_linkedin` and `draft_content`; this is the final stage.

1. Replace `{{LINKEDIN_URL}}` in the drafted Slack message with the real URL from `post_linkedin`. If the placeholder is missing, append `\n<URL>` so the link is always present.
2. Using `execute_python` with the Slack `vault_api_ids`, `POST https://slack.com/api/chat.postMessage` with JSON `{"channel": "<channel_id>", "text": "<message>", "unfurl_links": true}`. Check the JSON `ok` field.
3. If Slack returns `not_in_channel`, try `conversations.join` once for a public channel; if that is not permitted or the channel is private, report that the Slack app must be invited to the channel. Retry `chat.postMessage` once after a successful join. Do not post to LinkedIn again.
4. When `ok` is true, record the channel and the message `ts`.
5. Call `report_playbook_progress(step_id="post_slack", status="done")`.
6. Final answer to the user: the LinkedIn post URL as a clickable link, the exact text posted to LinkedIn, the Slack channel and the exact Slack message posted, and any assumptions or defaults used. If Slack failed but LinkedIn succeeded, give the LinkedIn URL first and the Slack error, and do not mark the run done.
