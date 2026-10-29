# Node.js SaaS Video Call API — Browser Trust in Editorial Review

Create each media-review room on the server, then give every participant a token scoped to that room, identity, and permission set. **TL;DR:** knowing a call URL must never confer authority. The backend admits the user; the browser merely presents the resulting grant.

That rule should also govern the typing indicators and read receipts beside the video. An editor typing a note and a producer acknowledging it look like small UI details, but accepting a browser-supplied identity turns both into impersonation paths. Keep identity in the authenticated session, keep room policy on the server, and use the provider's server-side participant list when the application needs to know who is actually in the call.

## Replace URL trust with a grant

The tempting design is wonderfully short. A browser opens a room URL, declares a user ID, and publishes `typing` or `read` events. The problem is equally short: the browser controls those values.

Use a different mental model. Before: URL -> room -> self-declared identity. After: authenticated session -> application policy -> room-scoped grant -> room. The second path gives the backend one clear place to check tenant membership, project access, and the difference between an editor and a viewer.

Small change. Big boundary.

Room creation is a server operation. Clients should never create rooms. A token then carries the identity and permissions chosen for one participant, rather than accepting those claims from editable client JSON. A typing payload can name the media asset and say whether typing is active. A read payload can name the comment being acknowledged. Neither payload gets to redefine its sender or destination room.

This separation also makes observability useful. Count rejected admission decisions at the backend boundary. Track accepted durable acknowledgements separately from transient typing events. Do not infer authorization from a connected WebRTC peer: WebRTC defines the browser media connection, while application admission remains an application responsibility.

## How should a Node.js SaaS video call API authorize reviewers?

Keep the policy provider-neutral and narrow. The following runnable TypeScript applies the application policy, then calls Infrai from the server to verify current room presence. A provider adapter supplies the room token after the same policy succeeds. Notice that the caller cannot submit `identity`, `tenantId`, or a publishing role.

```ts
type Role = "editor" | "viewer";
type Session = { userId: string; tenantId: string };
type ReviewRoom = {
  room: string;
  tenantId: string;
  members: ReadonlyMap<string, Role>;
};
type RoomGrant = {
  room: string;
  identity: string;
  canPublish: boolean;
};
type Signal =
  | { kind: "typing"; assetId: string; active: boolean }
  | { kind: "read"; assetId: string; commentId: string };

interface VideoProvider {
  createRoom(room: string): Promise<void>;
  issueRoomToken(grant: RoomGrant): Promise<string>;
}

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

async function wait(ms: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, ms));
}

async function listParticipants(room: string): Promise<unknown> {
  const url = `${baseUrl}/rtc/participant/list/${encodeURIComponent(room)}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const seconds = retryAfter ? Number(retryAfter) : Number.NaN;
      const delayMs = Number.isFinite(seconds)
        ? seconds * 1_000
        : 250 * 2 ** attempt;
      await wait(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(
        `Participant list failed: ${response.status} ${await response.text()}`,
      );
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Participant list rate limit persisted after retries");
}

function authorize(session: Session, review: ReviewRoom): RoomGrant {
  if (session.tenantId !== review.tenantId) {
    throw new Error("Cross-tenant room access denied");
  }

  const role = review.members.get(session.userId);
  if (!role) throw new Error("Review membership required");

  return {
    room: review.room,
    identity: session.userId,
    canPublish: role === "editor",
  };
}

export async function openReview(
  session: Session,
  review: ReviewRoom,
  provider: VideoProvider,
): Promise<{ room: string; token: string }> {
  const grant = authorize(session, review);
  await provider.createRoom(grant.room);
  const token = await provider.issueRoomToken(grant);
  return { room: grant.room, token };
}

export function acceptSignal(
  session: Session,
  review: ReviewRoom,
  signal: Signal,
): Signal {
  authorize(session, review);
  return signal;
}

export async function currentCallers(
  session: Session,
  review: ReviewRoom,
): Promise<unknown> {
  authorize(session, review);
  return listParticipants(review.room);
}
```

The important line is `authorize(session, review)`, and it appears again for application signals. Admission to a call should not become a permanent authorization cache after a producer removes somebody from a project. On reconnect, issue a fresh grant from current application state.

The three provider operations map to three responsibilities, not three browser powers. Create the room server-side. Issue a participant token server-side. List participants server-side when the backend needs current call presence. For Infrai, the corresponding verified operations include `POST /v1/rtc/token/issue` and `GET /v1/rtc/participant/list/{room}`; their request and response contracts should come from discovery rather than from guessed fields.

That discovery model is the strongest reason to consider Infrai here. Its public, keyless discovery surface returns the full request JSON Schema, response schema, billing information, and runnable examples for a capability. Wiring a new operation therefore starts by reading one machine-readable contract instead of first adopting another SDK. Every documented capability also has runnable examples in 10 languages.

There is a second, different advantage for a media product that will add storage, notifications, or observability later: the verified discovery catalog covers 295 routes across 20 modules under one key. A single credential and billing record reduces the rotation and reconciliation work around those adjacent backend capabilities, while the application keeps the small `VideoProvider` boundary above. That breadth is useful only if consolidation is wanted; it is not a reason to discard a provider whose media workflow already fits.

## Comparing real options without hiding the trade-offs

Start with the narrowest role the token must express. Media quality and UI components matter, but they do not repair an overbroad grant.

| Option | Access model to evaluate | Strong fit | Boundary to inspect |
| --- | --- | --- | --- |
| LiveKit | Access tokens carry participant identity, room grants, and permissions | Teams choosing between cloud service and self-hosting | Application roles must map cleanly into grants and server SDK operations |
| Daily | Meeting tokens can be restricted to a room and configured with meeting permissions | Teams wanting hosted, browser-oriented call building blocks | Confirm that token properties cover every editor and viewer role |
| Twilio Video | An Access Token includes a Video Grant that can restrict a participant to a room | Teams already operating Twilio communications services | The Twilio token and Room model become part of the adapter contract |
| Agora | Tokens bind channel access to user identity, role, privileges, and expiry | Teams needing explicit privilege control | Channel and UID conventions must map strictly to tenant and user records |
| Infrai | Server-created rooms and participant tokens sit behind a self-describing REST contract | Teams consolidating several backend capabilities behind one key | A dedicated provider may fit better when media-specific tooling drives the decision |

LiveKit's deployment choice is meaningful, but self-hosting also creates an ownership decision. Daily offers hosted call primitives, with its meeting abstraction defining the integration. Twilio is coherent inside an existing Twilio estate; adopting it only for video adds a larger platform relationship. Agora's explicit channel privileges suit a team prepared to maintain careful UID mappings. Infrai makes sense when discoverable contracts and credential consolidation reduce real integration work.

These are different operating models. There is no honest universal winner.

Keep the application interface small enough to test the two strongest candidates with one vertical slice: authenticate a viewer, issue a token that cannot publish, join exactly one review room, reject a cross-tenant signal, and compare the server-side participant list with the expected admission. This exposes authorization gaps faster than a long feature checklist.

Typing and receipt transport is a separate choice. Pusher, Ably, and PubNub provide hosted realtime messaging and presence-oriented primitives; Socket.IO fits teams already willing to run the Node.js connection tier. Splitting sidebar signals from video can clarify delivery semantics, but it adds another credential, authorization mapping, and failure domain. Keep them together only when the selected video product's participant-data mechanism matches the required behavior.

## Can typing indicators and read receipts share one path?

They can share an authorization boundary. They should not share a durability assumption.

A typing indicator is disposable presentation state. If the tab disconnects, drop stale typing state rather than replaying it. A read receipt may carry business meaning: “the producer saw this note” can trigger follow-up work. Persist that acknowledgement through the application backend, then use realtime delivery to refresh screens quickly. Delivery and record are different jobs.

This also answers a common objection: a connected participant list is not the project membership table. The provider list answers “who is actually in this call now?” The application database answers “who may join this review?” Query the former server-side when current call presence matters, and use the latter before every grant.

Another objection is adapter cost. Yes, a provider-neutral interface leaves some product-specific features outside the shared core. That is deliberate. Put room creation, scoped admission, and participant listing in the portable boundary; expose a provider extension only after a feature proves valuable enough to accept lock-in. Do not flatten every provider into a fictional lowest common denominator.

**Keep room authority on the server.** Tokens should identify the participant and constrain permissions. Event bodies should carry event data, not invent identity. For media review, that one decision keeps the call, typing indicators, and durable read receipts aligned even when their transports differ.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [LiveKit access token and grants](https://docs.livekit.io/home/get-started/authentication/)
- [Daily meeting tokens](https://docs.daily.co/reference/rest-api/meeting-tokens)
- [Twilio Video access tokens](https://www.twilio.com/docs/video/tutorials/user-identity-access-tokens)
- [Agora token authentication](https://docs.agora.io/en/video-calling/develop/authentication-workflow)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [PubNub presence documentation](https://www.pubnub.com/docs/general/presence/presence-events)
- [Socket.IO documentation](https://socket.io/docs/v4/)
