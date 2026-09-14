Project Title

Multi-Platform Conversational Publishing & Engagement Engine

Overview

This project automates the flow of media content from WhatsApp directly to Facebook and Instagram, and handles intelligent, context-aware replies to comments received on those platforms — all built using n8n as the orchestration engine, without writing a traditional backend application.

The core idea: a user sends a photo or video with a caption via WhatsApp. Within seconds, that same content is automatically published to a Facebook Page and an Instagram Business account with the identical caption. Separately, whenever someone comments on any of those posts, an AI model reads the comment, understands its context, and posts a unique, relevant reply — never a generic or repeated response.

Architecture Summary

The system consists of four independent n8n workflows:

WhatsApp (Evolution API) → Social Media Publisher — uses a self-hosted, unofficial WhatsApp automation layer
WhatsApp (Meta Cloud API) → Social Media Publisher — uses Meta's official, production-grade WhatsApp Business API
Facebook Comment Handler — listens for comments on Facebook posts and replies via AI
Instagram Comment Handler — listens for comments on Instagram posts and replies via AI

Both WhatsApp routes were built in parallel: the Evolution API route was built first as a proof-of-concept and remains functional, while the Meta Cloud API route was subsequently built to provide a production-stable, Meta-sanctioned alternative that does not risk account bans.

Route 1: WhatsApp (Evolution API) → Facebook + Instagram
Infrastructure
Docker Desktop running three containers locally: n8n, evolution-api (v2.3.7), and a PostgreSQL database for Evolution API's session storage.
Evolution API connects to a real WhatsApp number (923351657929) by scanning a QR code, effectively automating WhatsApp Web rather than using an official API.
n8n itself runs with an internal Docker sub-stack for its own JavaScript "Code" node execution — this is n8n's Task Runner architecture (separate runners, sandbox-api, and sandbox-cert containers), which caused significant early debugging (documented below).
Workflow: "WhatsApp Content Receiver"

Node chain:

Webhook (POST) 
  → Edit Fields (extracts sender, pushName, messageType, isFromMe, caption, mimetype, messageKey from the raw Evolution API payload)
  → HTTP Request (POST to Evolution API's /chat/getBase64FromMediaMessage/{instance} endpoint, authenticated via an apikey header, to fetch the media as a base64 string — Evolution API's webhook does not include base64 media inline; it only sends a media URL/reference, requiring this explicit follow-up call)
  → Convert to File (converts the base64 string into a binary file object usable by downstream HTTP nodes)
  → HTTP Request "Upload to Cloudinary" (POST to https://api.cloudinary.com/v1_1/{cloud_name}/auto/upload, multipart/form-data, fields: file, upload_preset — using an unsigned upload preset to avoid exposing API secrets)
  → If node (branches based on messageType: image vs. video)
  → [Image branch] HTTP Request "Post Image FB" → HTTP Request "Create IG Container" → HTTP Request "Publish IG Post"
  → [Video branch] HTTP Request "Post Video FB" → HTTP Request "Create IG Video Container" → HTTP Request "Check Video Status" (polls Instagram's Graph API for status_code) → Wait node → HTTP Request "Publish IG Video"
Key API Endpoints Used
Facebook Photo Post: POST https://graph.facebook.com/v21.0/{page-id}/photos (multipart form: url, caption, access_token)
Facebook Video Post: POST https://graph.facebook.com/v21.0/{page-id}/videos (multipart form: file_url, description, access_token)
Instagram Container Creation: POST https://graph.facebook.com/v21.0/{ig-business-account-id}/media (fields: image_url or video_url + media_type: REELS for video, caption, access_token)
Instagram Publish: POST https://graph.facebook.com/v21.0/{ig-business-account-id}/media_publish (fields: creation_id, access_token)
Credentials Configured
Facebook Page: "OptimusFox Pvt Ltd" (Page ID: 1218476514692879)
Instagram Business Account ID: 17841433751186422 (linked to the same Page)
Meta Developer App: "Social Media Automation" (App ID: 1481652093725885)
Cloudinary account: cloud name wxlwkofl, unsigned upload preset whatsapp_upload
Route 2: WhatsApp (Meta Cloud API, Official) → Facebook + Instagram

Built as a production-stable alternative because the Evolution API approach technically violates WhatsApp's Terms of Service (it automates WhatsApp Web without official sanction) and carries a real risk of the number being banned. The Cloud API route uses Meta's officially sanctioned Business messaging product.

Setup
A separate Meta Developer App was created: "OptimusFox Automation Tool" (App ID: 896759056633112), linked to Business Portfolio "OptimusFoxPvtLtdProfessionals" (Business ID: 1117675637461236). This was kept separate from the Facebook/Instagram posting app to isolate WhatsApp Business permissions cleanly.
A dedicated WhatsApp Business Account was registered: ID 1469073511704130, with production phone number +92 335 1657923, fully verified.
Since n8n is self-hosted on localhost:5678, and Meta requires a publicly reachable HTTPS callback URL for webhooks, ngrok was used to expose the local instance: static forwarding domain backstage-spectacle-unclaimed.ngrok-free.dev.
Workflow: "WhatsApp Cloud API Receiver"

Node chain:

Webhook (GET) — handles Meta's one-time verification handshake
  → Respond to Webhook (returns the value of the hub.challenge query parameter as plain text, which is required for Meta to confirm webhook ownership)

Webhook (POST, same path) — receives actual incoming message events
  → Code node (parses entry[0].changes[0].value from Meta's payload; extracts sender, messageType (msg.type — "image", "video", or "text"), mediaId, and caption; explicitly returns [] early for status-update-only payloads that lack a messages array, to avoid downstream errors)
  → HTTP Request (GET https://graph.facebook.com/v21.0/{{mediaId}} with access_token as a query parameter — NOT a header — to resolve the media ID into a temporary downloadable media URL)
  → HTTP Request (GET the resolved media URL to download the actual binary file, also authenticated with access_token as a query parameter)
  → Upload to Cloudinary → If (image/video split) → [same Facebook/Instagram posting pattern as Route 1]
Access Token Strategy

Initially, a temporary token (~24-hour expiry) generated from the App's "API Setup" page was used for testing. For production stability, this was replaced with a permanent System User token:

A System User ("OptimusFox Automation Bot") was created in Meta Business Settings under the relevant Business Portfolio, assigned the "Admin" role.
The System User was granted Full Access (Manage app) to the "OptimusFox Automation Tool" App, and Full Access to the WhatsApp Business Account.
A new token was generated for this System User with Token Expiration set to "Never", scoped with whatsapp_business_messaging and whatsapp_business_management permissions.
This permanent token replaced the temporary one in the access_token query parameter of both media-resolution HTTP Request nodes.
Route 3: Facebook Comment Auto-Reply
Workflow: "Facebook Comment Handler"

Node chain:

Webhook (GET) — Meta verification handshake
  → Respond to Webhook (returns hub.challenge)

Webhook (POST, same path) — receives feed change events
  → Code node — filters the payload to isolate genuine new comments (checks value.item === "comment" and value.verb === "add"), extracting comment_id, post_id, and comment_text (message field)
  → HTTP Request "Get Comment Sender" (GET /{comment_id}?fields=from&access_token=... — REQUIRED because Facebook's "feed" webhook field does NOT include the commenter's user ID inline, unlike Instagram's equivalent payload; this extra API call is the only way to identify who posted the comment)
  → If node — compares the fetched commenter ID against the Page's own ID; if they match (meaning the Page itself, or the bot's own reply, triggered this event), the workflow stops here — this prevents an infinite reply loop where the bot would otherwise reply to its own replies
  → Google Gemini node (model: gemini-3.6-flash) — given a system prompt describing the business context and instructed to produce a short, natural, contextually relevant reply in Roman Urdu/English mix, matching the tone of the original comment
  → HTTP Request — POST https://graph.facebook.com/v21.0/{comment_id}/comments (fields: message, access_token) to publish the AI-generated reply directly under the original comment

Note on AI provider: OpenAI's GPT API was the original plan, but was abandoned after hitting a billing/no-credits error on the account. Google Gemini (free tier) was substituted successfully with no functional loss.

Webhook Subscription Configuration (Critical Detail)

Meta's Webhooks product for the "Page" object exposes dozens of subscribable fields (about, birthday, feed, founded, etc.). The feed field specifically must be toggled to Subscribed for comment events to be delivered at all — this was initially missed and caused a completely silent failure (no executions triggered, no errors, nothing) until identified and corrected.

Additionally, subscribing the Page's webhook alone was insufficient — the Page itself had to be explicitly linked to the App via a POST /{page-id}/subscribed_apps?subscribed_fields=feed call in Graph API Explorer, which initially failed with a missing-permission error (pages_manage_metadata was not yet granted) and succeeded once that permission was added to the access token.

Route 4: Instagram Comment Auto-Reply
Workflow: "Instagram Comment Handler"

Follows the same architectural pattern as the Facebook handler, with two key differences:

Reply endpoint: Instagram uses POST /{comment_id}/replies (not /comments, as Facebook does).
Sender identification: Instagram's comments webhook field payload includes the commenter's ID directly (value.from.id) in the initial event, unlike Facebook — so the extra "Get Comment Sender" lookup step used in the Facebook workflow is unnecessary here, simplifying the chain slightly.

Instagram Business Account ID 17841433751186422 is shared with the posting workflows, as it is the same account.

Errors Encountered and Resolutions (Full Debugging Log)

This section documents every significant failure encountered during development, in chronological order, to demonstrate the debugging process.

1. n8n Task Runner Timeout ("Task request timed out")

Symptom: Every execution of any JavaScript "Code" node failed after a 60-second wait with a generic timeout error, with no clear cause.
Root cause: n8n's Docker deployment includes a separate internal container architecture (runners, sandbox-runner, sandbox-api, sandbox-cert) responsible for isolating JavaScript execution. The runners container's logs revealed "Task broker is down, launcher will try to reconnect" — a networking/connectivity failure between the main n8n container and its runner sidecar, unrelated to the workflow logic itself.
Resolution (interim): Replaced the "Code" node with an "Edit Fields (Set)" node using manual field mappings and n8n expressions — this bypasses the Task Runner entirely since Set nodes execute inside the core n8n process.
Resolution (permanent): Later, after the underlying Docker stack settled/restarted correctly, Code nodes were successfully reintroduced without recurrence.

2. Cloudinary "Bad request - Empty file"

Symptom: The Cloudinary upload step failed, reporting the file parameter as empty.
Root cause: The base64 field was not being extracted from the raw webhook payload at all in the original Code node — the extraction logic simply never referenced message.imageMessage.base64, so the field was always undefined.
Resolution: Updated the extraction logic to explicitly pull base64 from the correct nested path in the payload, with a fallback check across possible payload shapes (data.message.imageMessage.base64 vs data.message.base64), since Evolution API's payload structure can vary slightly.

3. Instagram "Media download has failed" / aspect ratio rejection

Symptom: Facebook photo posting succeeded, but the identical image was rejected when creating an Instagram media container.
Root cause: The source image's aspect ratio (1006×1512 px ≈ 0.665) fell outside Instagram's accepted range (minimum 4:5 = 0.8, maximum 1.91:1). Facebook's Photos API has no such restriction, which is why the same image succeeded there but failed on Instagram.
Resolution: Applied a Cloudinary URL transformation on-the-fly (c_fill,ar_4:5 inserted into the delivery URL) to automatically crop/reformat the image into Instagram's accepted aspect ratio before it is submitted to the Graph API — no manual image editing required.

4. Instagram video publish "Media ID is not available"

Symptom: Instagram video container creation succeeded, but the immediate "publish" call failed.
Root cause: Instagram processes uploaded video asynchronously in the background; a fixed-duration Wait node (30 seconds) was sometimes insufficient for longer videos, so the publish call fired before Instagram had finished transcoding.
Resolution: Introduced an explicit status-polling step — a GET request to /{container-id}?fields=status_code — checked before publishing, combined with an increased Wait duration, ensuring the video is confirmed FINISHED before the publish call is made.

5. Facebook OAuth "Invalid OAuth access token — Cannot parse access token"

Symptom: A previously working Facebook posting node suddenly failed.
Root cause: Hidden formatting artifacts (line breaks / invisible characters) were introduced into the access token when it was pasted into the node's parameter field.
Resolution: Cleared the field entirely and re-pasted the token as a single unbroken line, confirming no line wraps were present in the input box.

6. Facebook/Instagram Long-Lived Token Expiring Prematurely

Symptom: All Facebook/Instagram-related nodes across multiple workflows began failing simultaneously with "Error validating access token: Session has expired".
Root cause: The previously generated 60-day long-lived Page Access Token had expired (tokens generated through the interactive Graph API Explorer flow, rather than a System User flow, are more prone to premature invalidation from session-related events).
Resolution: Regenerated a fresh Page Access Token via Graph API Explorer, exchanged it for a new long-lived token via the oauth/access_token?grant_type=fb_exchange_token endpoint, and propagated the new token across every affected node (Facebook photo/video posting, Instagram container/publish, comment-sender lookup, comment reply posting).

7. WhatsApp Cloud API — Zero Executions Despite Correct Setup

Symptom: The webhook verification succeeded, the callback URL was confirmed reachable, yet sending real WhatsApp messages triggered no n8n executions whatsoever.
Root cause: On Meta's Webhook Configuration page for the WhatsApp product, the messages field — the one that actually carries incoming message events — was left in an Unsubscribed state among dozens of other available (and irrelevant) fields like account_alerts, calls, flows, etc.
Resolution: Explicitly toggled messages to Subscribed; executions began firing correctly immediately after.

8. IF Node Silently Routing Videos to the Image Branch

Symptom: After switching to the WhatsApp Cloud API route, videos were incorrectly being processed through the image-posting branch, causing Facebook to reject the file ("Photos should be less than 10 MB and saved as JPG, PNG..." when an MP4 was submitted).
Root cause: The IF node's condition used the .item accessor ($('Code in JavaScript').item.json.messageType) to reference a prior node's output. Because several HTTP Request nodes sat between the Code node and the IF node, n8n's paired-item tracking broke silently, causing the expression to resolve incorrectly without throwing a visible error.
Resolution: Replaced .item with .first() ($('Code in JavaScript').first().json.messageType), which retrieves the first output item directly by node reference rather than relying on paired-item chain tracking — resolved the misrouting immediately.

9. Gemini API "Too Many Requests" (Quota Exceeded)

Symptom: The AI reply-generation node intermittently failed during rapid, repeated manual testing.
Root cause: Google Gemini's free tier enforces a strict per-minute request quota; multiple test comments submitted within a short window exceeded it.
Resolution: No code change required — simply spacing out test requests by 1–2 minutes resolved it, as free-tier quotas reset on a rolling basis. Noted as a scaling consideration if comment volume increases in production (would require enabling billing).
