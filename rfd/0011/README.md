---
authors: Thomas Li
state: prediscussion
discussion:
labels: platform, media, infrastructure, interop
---

# [RFD] Real-Time Video Invigilation Using LiveKit

This RFD adds a `proctoring` extension (per [RFD 0009](../0009/README.md)) that uses [LiveKit](https://livekit.io) as the media transport for on-demand video invigilation of online onsite examinations, supporting up to 1,000 concurrent students per exam.

Scope is narrow: media transport, token issuance, and a full lifecycle from exam start to exam stop. Automatic room lifecycle, recording, alerting, and cross-extension choreography are deferred to a follow-up proctoring extension RFD.

## Background

Exams run in a supervised lab on a closed exam intranet that has no route to the public internet during the exam window. Each student publishes a **camera**, a **microphone**, and their **exam-machine screen** continuously; an invigilator spot-checks one student at a time and chooses which of those tracks to view. Pushing every track to every invigilator would saturate both directions of the network at 1,000-student scale, so we need a media topology where idle publications cost essentially nothing and bandwidth scales only with the number of active inspections. The intranet constraint also forces **self-hosting**: LiveKit Cloud is not reachable during exams and is not a candidate.

LiveKit is an open-source WebRTC SFU that provides the three primitives this requires:

- **Selective subscription** — clients can connect with `autoSubscribe: false` and subscribe to individual publications on demand via `setSubscribed`.
- **Publication metadata without subscription** — LiveKit keeps `TrackPublication` metadata (who is connected, camera on/off, muted) visible to every room member on the signalling channel, independent of subscription state. Invigilators can render a tile grid of all students at zero media cost.
- **Dynacast** — when no subscriber consumes a simulcast layer, the SFU signals the publisher to pause it. Combined with VP8 simulcast, an unwatched student's browser stops encoding video entirely; the uplink drops to signalling-only (~1–5 kbps).

## Scope

### In scope

- A `proctoring` extension (`ClientModule`) holding the LiveKit API secret and exposing routes for room create/delete, token issuance, and webhook ingestion.
- Authorization enforced twice: via OpenFGA in the extension's handler middleware, and via LiveKit's `VideoGrant` at the SFU.
- Publishing of camera, microphone, and screen from the student client, and on-demand inspection of any of those tracks from the invigilator client.
- New routes in the `ui-v2` staff and student apps, using a new `livekit-client` dependency.
- End-to-end lifecycle walkthrough from exam start to exam stop.

### Deferred to the proctoring extension RFD

- Automatic room create/destroy tied to `submission_collection.start_at`/`stop_at` (Temporal workflow per [RFD 0010](../0010/README.md)).
- A dedicated `can_proctor` relation in OpenFGA. For now this RFD reuses `coordinator from parent` on `activity`.
- Egress recording, invigilator-to-student audio warnings, multi-room sharding, `proctoring.*` NATS events, disconnect grace-period policy, attendance integration.

## Architecture

The extension is a standalone process that registers with `core` over NATS like any other extension; it receives broker/authn/authz/temporal config through `GetConfiguration()` and loads its own LiveKit config (host, API key, API secret) from YAML. `core` is unchanged. Frontend routes in `ui-v2` call the extension directly; OpenAPI specs from swaggo flow through Kubb into `packages/sdk` as usual.

```
ui-v2 staff/student apps ────────────────────── livekit-client
       │                                               │
       │ HTTPS /v1/proctoring/*                        │ wss (media + signalling)
       ▼                                               ▼
proctoring extension ──── NATS ──── core         LiveKit SFU
                                                 (self-hosted, VP8 simulcast + Dynacast)
```

### HTTP routes

| Method | Path                                                | Authorization                                     |
|--------|-----------------------------------------------------|---------------------------------------------------|
| POST   | `/v1/proctoring/rooms`                              | `can_edit` on activity                            |
| GET    | `/v1/proctoring/rooms?activity_id=<id>`             | `examinee` or `can_edit` on activity              |
| DELETE | `/v1/proctoring/rooms/:name`                        | `can_edit` on room's parent activity              |
| POST   | `/v1/proctoring/rooms/:name/tokens`                 | room membership (student) or `can_edit` (staff)   |
| POST   | `/v1/proctoring/webhooks/livekit`                   | LiveKit-signed (`Authorization` JWT, no session)  |

All routes except the webhook endpoint are mounted behind `RequireAuthenticated`. The webhook endpoint is verified using `webhook.ReceiveWebhookEvent` from `github.com/livekit/protocol/webhook`; the existing ghost-pipeline webhook in `core` (query-string token, no cryptographic verification) is explicitly not a template.

### Rooms and membership

One activity maps to **many rooms**, so large cohorts can be split into separate invigilation groups (e.g. by course section, physical lab, or staff-level grouping). The extension persists `(room_name, activity_id, participant_identities[])` tuples in-memory for each live room; the follow-up RFD will back this with a `proctoring_sessions` table.

Room creation takes an explicit name chosen by the staff caller (`POST /v1/proctoring/rooms` with body `{name, activity_id, participant_ids}`). The name is forwarded verbatim to `RoomServiceClient.CreateRoom` and must be unique across active rooms; collisions are rejected. `participant_ids` is the list of student user IDs allowed into this specific room.

Room-listing (`GET /v1/proctoring/rooms?activity_id=…`) returns the rooms the caller can join: for a staff user with `can_edit`, every room on the activity; for a student with `examinee`, only the rooms whose `participant_ids` contains their identity (normally exactly one). This is the student client's discovery step before requesting a token.

Token issuance authorization therefore has two layers for students: they must hold `examinee` on the parent activity (unchanged), **and** their identity must appear in the target room's `participant_ids`. Staff authorization stays at `can_edit` on the parent activity.

### Token grants

Tokens are short-lived (≤ 10 min) JWTs signed by the extension and fetched client-side at connection time — never embedded in SSR HTML. LiveKit's server auto-refreshes tokens on the signalling channel every ~10 min, so no client refresh polling is needed.

| Field                  | Student                | Invigilator     |
|------------------------|------------------------|-----------------|
| `RoomJoin`             | true                   | true            |
| `Room`                 | `<room name>`          | `<room name>`   |
| `CanPublish`           | **true**               | false           |
| `CanSubscribe`         | **false**              | **true**        |
| `CanPublishData`       | false                  | false           |
| `CanPublishSources`    | `[camera, microphone, screen_share]` | (n/a) |
| `CanUpdateOwnMetadata` | false                  | false           |

`CanSubscribe: false` is the **hard enforcement boundary** — the SFU rejects any subscribe attempt from an identity without the grant, regardless of client behaviour. `autoSubscribe: false` on the student client is a defensive supplement only. `CanUpdateOwnMetadata: false` blocks students from forging the publication metadata the invigilator grid reads. `screen_share_audio` is deliberately omitted from the student's `CanPublishSources`: system audio would leak OS notifications, ambient audio from other apps, and generally creates privacy exposure with no invigilation value the microphone doesn't already cover.

### Subscription model

All clients connect with `autoSubscribe: false`.

- **Students** publish three independent tracks on join — `camera`, `microphone`, and `screen_share` (`Track.Source` values `Camera`, `Microphone`, `ScreenShare`) — and never call `setSubscribed`.
- **Invigilators** render a tile grid from `room.remoteParticipants` and their `trackPublications`. Each student contributes three publication entries the grid can show state for (camera on/off, mic muted, screen share live or stopped); publication metadata is visible without subscribing. The grid stays current via `RoomEvent.ParticipantConnected`, `ParticipantDisconnected`, `TrackPublished`, `TrackUnpublished`, `TrackMuted`, `TrackUnmuted`.
- To spot-check a student, the invigilator picks any combination of their publications and calls `publication.setSubscribed(true)` + `publication.setVideoQuality(VideoQuality.HIGH)` on each. To stop: `publication.setSubscribed(false)`.

### Codec, Dynacast, simulcast

Student clients must publish **VP8** with simulcast and Dynacast:

```js
new Room({
  adaptiveStream: true,
  dynacast: true,                                         // off by default
  publishDefaults: { videoCodec: 'vp8', simulcast: true },
});
```

| Layer  | Resolution | Target     | Headroom  |
|--------|------------|------------|-----------|
| LOW    | 320 × 180  | ~100 kbps  | ~110 kbps |
| MEDIUM | 640 × 360  | ~400 kbps  | ~420 kbps |
| HIGH   | 1280 × 720 | ~1.5 Mbps  | ~1.6 Mbps |

**VP8 is required.** With SVC codecs (VP9/AV1), Dynacast can only pause whole streams, not individual simulcast layers, which breaks the idle-is-signalling-only property. With VP8 simulcast, each unused layer is paused by the SFU independently; when no invigilator is watching a student, every layer pauses and the browser stops encoding. Idle uplink collapses to RTCP keep-alives (~1–5 kbps). On resume the SFU requests a keyframe, producing a one-time burst at 10–20× steady-state bitrate — negligible unless subscriptions churn rapidly.

### Screen share

Screen is published as a separate track with `Track.Source.ScreenShare` using `room.localParticipant.setScreenShareEnabled(true)`. It uses VP8 + simulcast + Dynacast with independent configuration knobs (`publishDefaults.screenShareEncoding`, `publishDefaults.screenShareSimulcastLayers`), so an idle screen share pauses at the publisher the same way an idle camera does. Screen content is harder to compress than camera (sharp text, sudden full-frame changes on scroll), so bitrates are roughly 2× camera at the same resolution:

| Layer  | Resolution  | Target   | Headroom |
|--------|-------------|----------|----------|
| LOW    | 854 × 480   | ~200 kbps | ~250 kbps |
| HIGH   | 1280 × 720  | ~2.0 Mbps | ~2.5 Mbps |

Resolution is capped at 720p regardless of the student's display size — higher resolutions are wasted on invigilation and balloon the per-resume keyframe burst.

`getDisplayMedia` shows a native browser picker each time `setScreenShareEnabled(true)` is called; enforcing *which* surface the student picks is out of scope. Exams are onsite, invigilators are physically present, and the follow-up RFD will add session recording — screen-faking defences are not worth engineering against here. macOS requires a one-time Screen Recording permission per browser in System Settings, which belongs in pre-exam onboarding rather than the exam flow.

### Webhook events

| Event                             | Handler action                           |
|-----------------------------------|------------------------------------------|
| `room_started` / `room_finished`  | Open / close session record              |
| `participant_joined` / `_left`    | Record join/leave timestamps             |
| `participant_connection_aborted`  | Record abnormal disconnect (no alerting) |
| `track_published` / `_unpublished`| Log camera / mic / screen activity (distinguish via the payload's `source` field: `CAMERA`, `MICROPHONE`, `SCREEN_SHARE`) |

LiveKit retries on delivery failure, so handlers must be idempotent and dedupe on each payload's `event_id` (~5 min window).

## End-to-End Lifecycle

1. **Exam start.** Staff opens `/activities/:id/invigilate` in the staff app, configures one or more invigilation groups (name + participant set per group) and clicks **Start proctoring**. For each group, the client calls `POST /v1/proctoring/rooms` with `{name, activity_id, participant_ids}`. Extension checks `can_edit` on the activity, calls `RoomServiceClient.CreateRoom(name, EmptyTimeout=600)`, records the membership, returns `{room, wss_url}`.

2. **Student join.** Student opens `/activities/:id/proctored` in the student app. Client calls `GET /v1/proctoring/rooms?activity_id=<id>` to resolve the student's assigned room, then `POST /v1/proctoring/rooms/:name/tokens`. Extension verifies the caller's identity is in the room's `participant_ids`, issues a student grant, returns `{token, wss_url}`. Client constructs the `Room` (VP8 + simulcast + Dynacast), calls `room.connect(wss, token, { autoSubscribe: false })`, then `setCameraEnabled(true)`, `setMicrophoneEnabled(true)`, and `setScreenShareEnabled(true)`. The browser prompts for camera / microphone access once and for screen-share target on every `setScreenShareEnabled(true)` call. LiveKit fires `participant_joined` and one `track_published` per source.

3. **Invigilator join.** Staff picks a room from the rooms list and the client calls that room's token endpoint; extension issues an invigilator grant. Client connects with `autoSubscribe: false`, registers the `RoomEvent` listeners, and renders the tile grid from `TrackPublication` metadata. No video bytes flow. An invigilator monitoring multiple groups maintains one `Room` connection per group.

4. **Spot-check.** Invigilator clicks a tile and chooses which publications to view (camera, screen, microphone, or any combination) → `setSubscribed(true)` + `setVideoQuality(HIGH)` on each chosen publication. Dynacast resumes the HIGH layer on that publisher only; first frame arrives within the latency budget. Clicking away → `setSubscribed(false)` and the SFU re-pauses the layers.

5. **Transient disconnect.** Browser refresh fires `participant_left` then `participant_joined` a few seconds apart. Both logged, no alerting.

6. **Student finish.** Closing the exam tab fires `track_unpublished` then `participant_left`.

7. **Exam stop.** Staff clicks **Stop proctoring**; the client calls `DELETE /v1/proctoring/rooms/:name` for every room on the activity. Extension calls `RoomServiceClient.DeleteRoom` for each. LiveKit disconnects remaining participants and fires `room_finished` per room. Extension drops each room's membership record.

## Implementation Notes

- **Token endpoint rate limiting.** The `POST /v1/proctoring/rooms/:name/tokens` endpoint should be rate-limited per user per room to contain probing and accidental client loops. Room names are staff-chosen so guessability is a deployment concern, not an API one — but membership enforcement at the extension already prevents a correctly-guessed name from yielding a usable token.
- **Thumbnail grid (stretch).** Subscribing to every student's camera at `VideoQuality.LOW` is trivially implementable but crosses 80 Mbps downlink at 1,000 students; adding screen at LOW doubles it. If either is included, the invigilator client must enforce a hard cap architecturally via a fixed-size subscription pool (≤ 25 camera thumbnails, ≤ 6 screen thumbnails) — not as a UI hint.
- **Screen share re-prompt on page refresh.** Browser screen-share permission is per-capture and not remembered across page loads. On `Room` construction the client should check whether it previously had a screen track active (persisted to `sessionStorage` when `setScreenShareEnabled(true)` succeeded) and, if so, re-invoke `setScreenShareEnabled(true)` to surface the picker again. WebSocket-only reconnects are handled transparently by the SDK because the underlying `MediaStreamTrack` is preserved.
- **Secret handling.** The LiveKit API secret must never be logged, returned by any diagnostic endpoint, or committed as YAML outside development. Source it from environment or a secret manager.

## Open Questions

1. **Load test as sign-off blocker.** 1,000-participant publish-only topology with aggressive Dynacast has not been validated for ZINC. A `livekit-cli load-test` at 500 / 750 / 1,000 participants must measure SFU CPU + memory, connection-establishment latency, and TURN relay cost before this RFD moves to `published`.

2. **Mid-session permission revocation.** If a staff user's `coordinator` relation is revoked mid-exam, their SFU token remains valid until its ≤ 10 min TTL. Closing this gap requires `UpdateParticipant` or `RemoveParticipant` keyed on a NATS event from `core`'s authz layer. **Deferred**; the TTL window is an accepted gap here.

3. **Metadata visibility within a room.** `TrackPublication` and participant events fan out to every room member regardless of `CanSubscribe`, so every student in a given room learns the identities and display names of the other students in that room. Splitting cohorts into smaller rooms (above) bounds the exposure to group size rather than cohort size. Further mitigations — client-side filtering of non-self events — belong in the proctoring extension RFD. Accepted privacy trade-off for now.

4. **`coordinator from parent` escalation.** Any course coordinator — including TAs — inherits invigilator access under the interim relation reuse. Deployments should audit coordinator grants on examination activities until the dedicated `can_proctor` relation lands.

5. **Reconnect under long exams.** NAT binding timeouts (5–15 min) and TURN credential rotation can silently drop students during a 2–3 hour exam. The LiveKit SDK reconnects transparently in most cases, but a heartbeat strategy and a grace-period alerting policy belong in the proctoring extension RFD.


## References

- [RFD 0009 — ZINC Extension System](../0009/README.md)
- [RFD 0010 — Temporal Workflow Orchestration](../0010/README.md)
- LiveKit docs: [selective subscription](https://docs.livekit.io/home/client/tracks/subscribe/), [Dynacast + simulcast](https://docs.livekit.io/home/client/tracks/advanced/), [token generation](https://docs.livekit.io/home/server/generating-tokens/), [managing participants](https://docs.livekit.io/home/server/managing-participants/), [webhooks](https://docs.livekit.io/home/server/webhooks/)
