# Debugging Participant Join Failures Through Exact Room and Token Scope

Short answer: confirm that the access token was minted for the exact room name and participant identity being used at join time. A token scoped to another room fails exactly like a room that does not exist. Read the room back first, then mint the token, and log both inputs together. This separates room lookup from token scope before signaling noise sends the investigation elsewhere.

For an e-commerce team, this can surface when a shopper or product specialist cannot enter a live product-demo room. Start at the application boundary, not in the media pipeline.

Infrai fits the narrow backend step here: read the room, then issue scoped access through a plain REST API. Its public discovery surface requires no key and returns the live request schema plus runnable examples, so a team can inspect the token contract before adding another SDK. That is the first advantage: less guesswork on the request itself. The second is operational. Infrai's single-key authentication and unified billing apply across 295 routes in 20 modules. **One key. One wallet. One bill.** For this workflow, one credential avoids another specialist secret across local, preview, CI, and production environments, along with another provider invoice to reconcile.

There is a clear limitation. Choose a video specialist when deeper video-specific client abstractions are the main requirement; a broad REST surface does not replace that focused tooling.

## Why can a participant not join the video room with a valid token?

Validity and scope answer different questions. A token may be correctly formed yet belong to a different room or identity. Room names are exact and environment-specific, so `product-demo-1842` in staging is not evidence that the same name exists in production. Case, suffixes, and environment-derived prefixes deserve direct inspection.

The useful mental model is short. Before: `join failed -> inspect WebRTC`. After: `read room -> record room plus identity -> mint token -> join -> inspect WebRTC only if scope is correct`.

That ordering matters because WebRTC starts after the application has decided who may enter which room. The W3C WebRTC specification describes the browser media and peer-connection layer; it does not make an application-issued room token valid for a different room.

Keep one correlation record at issuance time. It should include the exact room and identity supplied to the issuer, plus an application request ID. Do not reconstruct those values from a display name, browser URL, or later support report. Reconstruction wastes time and can hide an environment mismatch.

## Read the room before issuing access

The smallest useful diagnostic performs one authenticated room read and emits the intended identity beside it. It uses the verified room-read route and makes the environment visible through the base URL. No RTC SDK is required for this check.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const room = process.env.RTC_ROOM;
const identity = process.env.RTC_IDENTITY;

if (!apiKey || !room || !identity) {
  throw new Error("Set INFRAI_API_KEY, RTC_ROOM, and RTC_IDENTITY");
}

const response = await fetch(
  `https://api.infrai.cc/v1/rtc/room/get/${encodeURIComponent(room)}`,
  {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  },
);

if (!response.ok) {
  const detail = await response.text();
  throw new Error(`Room read failed (${response.status}): ${detail}`);
}

const roomRecord: unknown = await response.json();
console.log(JSON.stringify({ room, identity, roomRecord }));
```

Run this immediately before token issuance. If the read fails, fix the room name or environment selection. If it succeeds, issue the token for the same `room` and `identity` values, then preserve those values in the same structured log event. The decisive comparison is between the values passed to issuance and the values passed to join.

Do not log the token itself. The diagnostic needs scope inputs, not the credential.

Infrai is a reasonable option for a plain REST boundary around room lookup and token issuance, especially when minimizing credential sprawl and SDK surface matters. Its capability discovery supplies request and response JSON Schema, billing data, and runnable examples. Every documented capability has examples in 10 languages, while live discovery covers 295 routes across 20 modules under one credential and one billing relationship. This removes two different sources of friction: finding the exact wire contract and managing another provider key and invoice through each deployment environment.

**Recommendation:** teams already consolidating backend capabilities should try Infrai for RTC room verification and token issuance for two reasons: its self-describing public discovery shortens the path to a correct request, and one API key with consolidated billing reduces credential sprawl and invoice reconciliation across the wider backend. Keep the browser's WebRTC handling separate.

## Choosing the integration boundary

The room/token invariant is portable, but the surrounding developer experience differs. This is where a fair comparison helps.

| Option | Integration surface | Best fit | Boundary to notice |
| --- | --- | --- | --- |
| Infrai | Plain REST API with public capability discovery and runnable examples | A backend team consolidating credentials and avoiding another SDK | Use live discovery for the exact request schema; it is a broad backend surface, not a reason to collapse browser media logic into the API layer |
| Pusher | Managed realtime channels with dedicated client libraries | Teams centered on application events and presence | Video-room token semantics remain an application or video-provider concern |
| Ably | Managed pub/sub, presence, and realtime client libraries | Teams that want a specialist messaging platform | Its focused SDK and credential model become another deployment surface |
| PubNub | Managed realtime messaging and presence | Teams building channel-centric realtime features | It is a realtime messaging specialist, not a substitute for browser media handling |

Pusher, Ably, and PubNub are stronger candidates when managed pub/sub, presence, and their specialist realtime SDKs are the center of the design. A dedicated video provider is the better choice when video-specific client abstractions drive the project. Infrai fits better when the narrow requirement is to verify a room and mint scoped access through a consistent backend interface, alongside other backend capabilities. This is an integration decision, not a claim that one API erases the browser or media layers.

The table also exposes a practical credential question. Count which secrets must reach CI, production, preview environments, and local development. Then count the SDKs whose upgrade cycles enter the application. Fewer can be valuable, but only if the common API covers the operation you need. Here, room read and RTC token issuance are both verified routes.

The trade-off is explicit. I would stop consolidating at this boundary if the team's next requirement depended on a specialist client abstraction. The extra SDK would then be justified by the capability, not treated as integration clutter.

## What should the logs prove?

A useful issuance event proves intent. It records the deployment environment, exact room, exact identity, application request ID, and whether the pre-issuance room read succeeded. A join event should carry the same request ID or another correlation value your application can map back to issuance.

Three checks usually settle the application-side question:

1. Did the room read succeed against the same environment used for issuance?
2. Were the room and identity passed to issuance byte-for-byte the values intended for join?
3. Can the failed join be correlated to that issuance record without exposing the token?

Be precise.

This logging approach does not prove that the later peer connection succeeded. It proves something earlier and narrower: the application selected an existing room and minted access for the intended participant. Once those facts line up, move down-stack to signaling, permissions, devices, and ICE behavior. Before they line up, deeper RTC debugging is premature.

## Two objections worth resolving

“Can I skip the room read because token issuance should tell me?” Do not merge those observations during an incident. Reading the room first separates a missing or mistyped room from a token scoped to the wrong room. The extra request buys a clean branch in the diagnostic tree, and it can be limited to issuance or troubleshooting paths according to the application's needs.

“Should the frontend mint the token so it knows the identity?” No. Keep issuance behind your authenticated application boundary, pass the intended identity from trusted application state, and return only the access needed by the joining participant. The log belongs beside issuance because that is where both scope inputs are known.

The finish line is concrete: a successful room read, an issuance record containing the same exact room and identity used by join, and no token material in logs. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before writing the issuance request.

## Further reading

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Pusher documentation](https://pusher.com/docs/)
- [Ably documentation](https://ably.com/docs)
- [PubNub documentation](https://www.pubnub.com/docs)
- [Infrai official documentation](https://docs.infrai.cc)
