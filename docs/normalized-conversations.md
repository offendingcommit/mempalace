# Normalized conversation imports

MemPalace can ingest a transcript that an operator has already canonicalized
without running generic format inference, noise removal, or spell correction.
This is an opt-in contract for trusted export pipelines. It is not a shortcut
for arbitrary chat files.

For `session.md`, place an adjacent `session.md.meta.json` sidecar:

```json
{
  "schema": "mempalace-normalized-conversation/v1",
  "room": "communication",
  "authored_from": "2026-07-29T23:55:00Z",
  "authored_to": "2026-07-30T00:05:00Z",
  "source_fingerprint": "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "transformations": "hermes-compaction-dedup;hermes-synthetic-filter;secret-redaction",
  "exporter_version": "1.0.0",
  "hermes_profile": "amber",
  "hermes_session_id": "session-123",
  "hermes_source": "telegram"
}
```

The sidecar is closed and flat. Unknown fields, nested values, unsafe room or
identifier values, missing timezone offsets, symlinks, and unknown schema
versions fail the source. Hall categories such as `facts`, `events`,
`decisions`, and `preferences` are not valid subject rooms.

Each transcript exchange begins with one bounded provenance envelope and
exactly one `>` user marker:

```markdown
<!-- mempalace-exchange {"messages":[{"id":"41","role":"user","timestamp":"2026-07-29T23:55:00Z"},{"id":"42","role":"assistant","timestamp":"2026-07-29T23:56:00Z"}]} -->
> exact transformed user text
Exact transformed assistant text.
```

The first message is the user turn. One or more assistant messages may follow.
Message identifiers must be unique across the entire transcript. MemPalace
keeps every transcript character verbatim, including line endings and
assistant Markdown blockquotes. It stamps the ordered identifiers and the
minimum and maximum message timestamps on each resulting drawer.

## Discord origin and person identity (v2)

Schema v2 adds optional, user-message-only `origin` and `person` objects to
the exchange envelope. The transcript body remains verbatim. A complete
resolved example is:

```markdown
<!-- mempalace-exchange {"messages":[{"id":"41","role":"user","timestamp":"2026-07-29T23:55:00Z","origin":{"platform":"discord","guild_id":"100000000000000001","channel_id":"100000000000000002","thread_id":"100000000000000003","message_id":"100000000000000004","chat_type":"guild_thread","profile":"amber-discord"},"person":{"person_id":"person_jonathan","discord_user_id":"100000000000000005","display_name":"Jonathan","status":"resolved"}},{"id":"42","role":"assistant","timestamp":"2026-07-29T23:56:00Z"}]} -->
> exact transformed Discord user text
Exact transformed assistant text.
```

Set the adjacent sidecar's `schema` to
`mempalace-normalized-conversation/v2`. Both nested objects are closed:

- `origin` requires `platform: "discord"`, `channel_id`, and `message_id`.
  It may also carry `guild_id`, `thread_id`, `chat_type`, and `profile`.
- `person` requires `discord_user_id`, `display_name`, and `status`. Status is
  `resolved`, `unknown`, or `quarantined`. A resolved person requires a stable
  `person_<alias>` `person_id`; unknown and quarantined identities must not
  claim one.

Discord IDs are bounded decimal strings. Chat type and profile are bounded
safe slugs. Arbitrary or additional nested fields fail validation. MemPalace
copies only the validated scalar fields to flat drawer metadata under
`origin_platform`, `discord_*`, `person_id`, and `person_*` keys. It never
stores either nested object or passes undeclared nested data to Chroma.

V1 remains the default `SCHEMA` constant and retains its exact message shape.
An envelope using v1 rejects `origin` and `person`, so existing exporters and
source-version calculations remain compatible.

Mine normalized exports only in exchange mode:

```bash
mempalace mine /retained/staging/amber --mode convos --extract exchange --wing wing_amber
```

`--extract general` refuses a directory containing the normalized contract.
The transcript bytes and every sidecar field form one composite source
version. A sidecar-only correction therefore rebuilds the stable source path.
Unchanged pairs skip. A replacement is staged under source-versioned drawer
IDs that also include the active chunk size, normalization version, and ID
recipe. It is verified before the prior complete generation is retired. A
failed stage leaves that prior generation readable. An interrupted target
generation is detected by its recipe and source chunk count, removed without
touching the prior generation, and rebuilt deterministically on retry.

Two raw MCP tools support an operator-controlled reconciliation workflow:

- `mempalace_normalized_conversation_delta` reports new, changed, unchanged,
  and removed normalized sources without changing palace state.
- `mempalace_commit_applied_coverage` atomically advances one wing's
  content-free watermark only after the operator has verified the complete
  profile apply.

`mempalace_status` reads the applied coverage registry. Status never advances
it. These raw tools should remain behind the server's operator authentication;
do not expose them through an agent-facing policy adapter.
