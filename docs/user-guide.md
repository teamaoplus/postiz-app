# User guide

Postiz schedules posts to social media and chat channels. You write a post once, pick the channels and a time, and Postiz publishes it in the background at that time.

## 1. Connect your channels

1. Open **Calendar** (the `/launches` page) and click **Add Channel**.
2. Pick a provider and complete its login (OAuth) flow. Providers that run on a self-hosted URL (WordPress, Mastodon, Lemmy, Listmonk, Bluesky PDS, ...) also ask for the URL of your instance.
3. The channel then shows in the channel list on the left. From its menu you can set preferences, move it to a customer, disable it, or remove it.

Supported providers include X, LinkedIn (profiles and pages), Facebook, Instagram, Threads, YouTube, TikTok, Pinterest, Reddit, Bluesky, Mastodon, Discord, Slack, Telegram, Google Business Profile, Dribbble, Tumblr, Medium, Dev.to, Hashnode, WordPress, Lemmy, Nostr, Farcaster, VK, Twitch, Kick, Skool, Whop, Listmonk and more.

> On a self-hosted install, a provider only works once its API credentials are set in `.env` (for example `LINKEDIN_CLIENT_ID` / `LINKEDIN_CLIENT_SECRET`). See [Development setup](./development.md).

### Customers

If you manage channels for several clients, group them by **customer**. Selecting a customer in the calendar filters the channels and posts down to that customer.

## 2. Create a post

1. In the calendar, click **Create Post**, or click an empty slot to start a post at that time.
2. Select one or more channels at the top of the editor.
3. Write the content in the global editor. It's shared by every selected channel.
4. To change the text for one channel, select that channel and edit its own version. Its settings tab also holds provider-specific options (a title for YouTube, a subreddit for Reddit, a board for Pinterest, ...).
5. Add media from the media library, upload new files, or generate an image or video with AI.
6. To post a thread or a follow-up comment, add more values below the first one. Each value can wait a delay after the previous one.
7. Check the preview, then choose:
   - **Add to calendar**: schedule it for the selected date and time.
   - **Post now**: publish right away.
   - **Save as draft**: keep it on the calendar without publishing.

Postiz validates every channel before scheduling, for example the character limit, the media each provider requires, and settings that must be filled in. Drafts skip those checks.

### Other editor options

- **Repeat**: republish the same post at a fixed interval.
- **Tags**: color-code posts in the calendar.
- **Signatures**: insert a saved text block. Manage them in Settings → Signatures.
- **Sets**: saved channel and settings combinations you can start a post from. Manage them in Settings → Sets.
- **Short links**: links can be shortened automatically. The default lives in Settings → Global Settings.

## 3. Work with the calendar

- Switch between day, week, month and list views.
- Drag a post to another slot to reschedule it.
- Click a post to edit, duplicate, preview or delete it, or to see its statistics.
- Posts that failed to publish are marked in the calendar and listed in the notifications bell, with the provider's error.

## 4. Media library

**Media** (top menu) lists every file uploaded to the organization. Upload images and videos here or from the post editor, and reuse them across posts. The **Integrations** page (third-party) connects external media tools, such as AI video generators, whose output also lands in the library.

## 5. Analytics

**Analytics** shows per-channel statistics, such as followers, impressions and engagement, for the providers that expose them. Pick a channel and a date range. Per-post statistics open from the post's menu in the calendar.

## 6. AI agent

The **Agent** page is a chat assistant that works with your account. It can list your channels, write and schedule posts, generate images and videos, and [clip YouTube videos](./clipping.md) when that feature is enabled. You can connect the same tools to external AI apps (Claude, ChatGPT, Cursor, ...) over MCP. See [Public API → MCP](./public-api.md#mcp).

## 7. Automations

- **Plugs** (top menu): per-channel automations, for example on X: repost a post, or add a promotional follow-up to it, once it reaches a number of likes. The plugs available depend on the provider.
- **Auto Post** (Settings → Auto Post): watch an RSS feed and create posts for its new items on the selected channels.
- **Webhooks** (Settings → Webhooks): get an HTTP call when posts are published.

## 8. Team and settings

Settings tabs (some only appear for admins or on plans that include them):

| Tab | Use it to |
| --- | --- |
| Global Settings | Profile, short-link preference, email notifications |
| Teams | Invite members by email with a **User** or **Admin** role, and remove them |
| Webhooks | Notify external systems when posts are published |
| Auto Post | RSS-to-post automations |
| Sets | Saved post presets |
| Signatures | Reusable text snippets for the editor |
| Developers | Your public API key, and MCP and CLI connection instructions |
| Approved Apps | OAuth apps you authorized to access your account, which you can revoke |

**Billing** (cloud only) shows your plan and its limits: channels, posts per month, AI image and video credits, and clipping minutes.
