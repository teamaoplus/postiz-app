# Public API

The public API lets other systems create and manage posts in a Postiz organization. It's implemented in `apps/backend/src/public-api/routes/v1/public.integrations.controller.ts`.

Official clients: [NodeJS SDK](https://www.npmjs.com/package/@postiz/node), [n8n node](https://www.npmjs.com/package/n8n-nodes-postiz), [Make.com](https://apps.make.com/postiz), and the [Postiz agent CLI](https://github.com/gitroomhq/postiz-agent).

## Base URL

```
{NEXT_PUBLIC_BACKEND_URL}/public/v1
```

For Postiz cloud, use the URL shown in **Settings → Developers**.

| Install | Base URL |
| --- | --- |
| Docker image (default compose) | `http://localhost:4007/api/public/v1` |
| Local dev | `http://localhost:3000/public/v1` |

## Authentication

1. In Postiz, open **Settings → Developers** and copy the API key.
2. Send it as-is in the `Authorization` header. There's no `Bearer` prefix:

```bash
curl -H "Authorization: YOUR_API_KEY" http://localhost:3000/public/v1/is-connected
# {"connected":true}
```

Tokens issued to OAuth apps (prefixed `pos_`) are accepted in the same header. On installs with Stripe billing, the organization needs an active subscription, otherwise every call returns `401 No subscription found`.

| Status | Body | Meaning |
| --- | --- | --- |
| 401 | `{"msg":"No API Key found"}` | Missing header |
| 401 | `{"msg":"Invalid API key"}` | Unknown key |
| 401 | `{"msg":"Invalid OAuth token"}` | Unknown `pos_` token |

## Rate limit

`POST /posts` is limited per organization per hour: 90 by default, or the value of `API_LIMIT` on self-hosted installs. Other endpoints aren't throttled.

## Typical flow

1. `GET /integrations` to find the channel IDs.
2. Optionally `POST /upload` or `POST /upload-from-url` to host media.
3. Optionally `GET /find-slot/:integrationId` to get the next free time slot.
4. `POST /posts` to create the post.

## Endpoints

### Channels

| Method | Path | Description |
| --- | --- | --- |
| GET | `/integrations?group={customerId}` | List channels: `id`, `name`, `identifier` (provider), `picture`, `disabled`, `profile`, `customer` |
| GET | `/groups` | List customers (`id`, `name`) to filter channels by |
| GET | `/integration-settings/:id` | Settings schema and rules for a channel's provider |
| GET | `/social/:provider?refresh=` | Get an OAuth URL that connects a new channel of that provider |
| DELETE | `/integrations/:id` | Remove a channel |
| POST | `/integration-trigger/:id` | Call a provider helper method: `{ "methodName": "...", "data": { ... } }` |
| GET | `/is-connected` | Check the key, returns `{ "connected": true }` |

### Posts

| Method | Path | Description |
| --- | --- | --- |
| GET | `/posts?startDate=&endDate=&customer=` | List posts in a date range (ISO dates) |
| POST | `/posts` | Create posts (see below) |
| DELETE | `/posts/:id` | Delete a post and the rest of its group |
| DELETE | `/posts/group/:group` | Delete a post group |
| PUT | `/posts/:id/status` | `{ "status": "draft" \| "schedule" }` |
| PUT | `/posts/:id/settings` | Update a post's provider settings |
| GET | `/posts/:id/missing` | Find posts on the provider when Postiz is missing the release ID |
| PUT | `/posts/:id/release-id` | `{ "releaseId": "..." }` sets the provider's post ID by hand |
| GET | `/find-slot/:integrationId` | Next free time slot: `{ "date": "..." }` |

### Media

| Method | Path | Description |
| --- | --- | --- |
| POST | `/upload` | `multipart/form-data` with a `file` field |
| POST | `/upload-from-url` | `{ "url": "https://..." }` |

Both return the stored media record. Use its `id` and `path` in a post's `image` array. When the server sets `RESTRICT_UPLOAD_DOMAINS`, posts can only use media uploaded through these routes.

### Analytics and other endpoints

| Method | Path | Description |
| --- | --- | --- |
| GET | `/analytics/:integrationId?date=` | Channel analytics |
| GET | `/analytics/post/:postId?date=` | Post analytics |
| GET | `/notifications` | Organization notifications, including publishing errors |
| POST | `/generate-video` | Start an AI video generation |
| GET | `/debug/posts/:id`, `/debug/account`, `/debug/channels`, `/debug/activity` | Troubleshooting views of a post timeline, the account, channel health and recent activity. **Super-admin keys only.** |

## Creating posts

`POST /posts`

```json
{
  "type": "schedule",
  "date": "2026-10-10T09:00:00.000Z",
  "shortLink": false,
  "tags": [],
  "posts": [
    {
      "integration": { "id": "CHANNEL_ID" },
      "value": [
        {
          "content": "Hello from the Postiz API",
          "image": [{ "id": "MEDIA_ID", "path": "https://.../image.png" }]
        },
        {
          "content": "A second value is a thread reply or comment",
          "delay": 0,
          "image": []
        }
      ],
      "settings": { "who_can_reply_post": "everyone" }
    }
  ]
}
```

| Field | Notes |
| --- | --- |
| `type` | `draft`, `schedule`, `now` or `update` |
| `date` | ISO date-time. When to publish. |
| `shortLink` | Shorten the links in the content |
| `tags` | `[{ "value": "...", "label": "..." }]` |
| `posts[]` | One entry per channel |
| `posts[].value[]` | The first value is the post. Every next value is a thread reply or comment, published `delay` after the previous one. |
| `posts[].value[].image[]` | Media from the upload routes: `id`, `path`, and optional `alt` and `thumbnail` |
| `posts[].settings` | Provider-specific settings. Postiz fills in `__type` from the channel. `GET /integration-settings/:id` shows what each provider expects (the example above is X). |
| `posts[].group` | Optional. Posts that share a group are edited and deleted together. |
| `republish` | `true` to publish an already-published post again |

The response is one entry per channel:

```json
[{ "postId": "...", "integration": "CHANNEL_ID" }]
```

### Validation

Non-draft posts go through the same checks as the editor: provider settings, character limits, and required media. A failure returns `400` with the channel and the reason, for example `post is too long, please fix it`. Drafts only need some text or one image.

## MCP

The same account is reachable from AI apps (Claude, ChatGPT, Cursor, VS Code, Codex, ...) over MCP. **Settings → Developers** generates the exact config for each client. The server URLs are:

| Auth | URL |
| --- | --- |
| OAuth (preferred) | `{backend}/mcp-oauth-dynamic` |
| API key in the URL | `{backend}/mcp/{API_KEY}` |
| API key in a header | `{backend}/mcp` with `Authorization: Bearer {API_KEY}` |

Example for Claude Code:

```bash
claude mcp add postiz --transport http "http://localhost:3000/mcp-oauth-dynamic"
```

MCP tools cover listing channels, scheduling posts, uploading media, generating images and videos, and [video clipping](./clipping.md).
