# Environment variables

How to fill in every variable in the `postiz` service of the compose file: what it does, whether you need it, and where to get the value.

We run Postiz with **Podman**, using `podman-compose.prod.yaml`. It is the same as `docker-compose.yaml` except that every image name is fully qualified (`docker.io/library/postgres:17-alpine` instead of `postgres:17-alpine`), because Podman does not assume Docker Hub for short image names. Edit the variables in `podman-compose.prod.yaml`.

Only the **Required settings** must be filled in. Everything else is optional: leave a provider's variables empty and that channel simply won't appear when you add a channel.

After changing any value, recreate the container so it picks it up:

```bash
podman-compose -f podman-compose.prod.yaml up -d --force-recreate postiz
```

In the examples below, `FRONTEND_URL` is `http://localhost:4007`. Replace it with your own URL (for example `https://postiz.example.com`).

## Running with Podman

| Task | Command |
| --- | --- |
| Start everything | `podman-compose -f podman-compose.prod.yaml up -d` |
| Stop everything | `podman-compose -f podman-compose.prod.yaml down` |
| Pull newer images | `podman-compose -f podman-compose.prod.yaml pull` |
| Check containers | `podman ps` |
| Follow Postiz logs | `podman logs -f postiz` |
| Check the variables the container actually got | `podman exec postiz printenv \| sort` |

`podman compose` (with a space) also works, but on this machine it hands off to the `docker-compose` plugin; `podman-compose` is the native tool.

Podman-specific notes:

- **Start on boot:** Podman has no always-running daemon, so `restart: always` does not bring containers back after a reboot by itself. Enable the restart service once: `systemctl --user enable --now podman-restart.service` (rootless) or `sudo systemctl enable --now podman-restart.service` (rootful). For rootless containers to start without anyone logged in, also run `sudo loginctl enable-linger $USER`.
- **Ports below 1024:** rootless Podman cannot bind ports such as 80 or 443. Keep Postiz on `4007` and put a reverse proxy in front for HTTPS, or lower the limit with `sudo sysctl net.ipv4.ip_unprivileged_port_start=80`.
- **Short image names:** if you add a service, write its image fully qualified (`docker.io/...` or `ghcr.io/...`), or Podman will prompt or fail when pulling.
- **Secrets in the compose file:** `podman-compose.prod.yaml` holds real values such as `JWT_SECRET`. Don't commit it to git once you fill in API keys.

### Using a LAN IP address

If `FRONTEND_URL` is a private address such as `http://192.168.x.x:4007`, connecting channels still works, because the OAuth redirect happens in your own browser. But platforms that download your media from Postiz (Instagram, Threads, TikTok, Pinterest and others) cannot reach a private IP, so posts with images or video to them will fail. For those, Postiz needs a public HTTPS domain, or use Cloudflare R2 storage so media is served from a public bucket.

## Required settings

| Variable | What to set |
| --- | --- |
| `MAIN_URL` | The public URL people open Postiz at, e.g. `https://postiz.example.com`. |
| `FRONTEND_URL` | Same as `MAIN_URL`. Every OAuth callback is built from it, so it must be exactly the URL in the browser (scheme, host, port, no trailing slash). |
| `NEXT_PUBLIC_BACKEND_URL` | `FRONTEND_URL` + `/api`, e.g. `https://postiz.example.com/api`. |
| `JWT_SECRET` | A long random string, unique per install. Generate one with `openssl rand -base64 48`. Changing it later logs everyone out. |
| `DATABASE_URL` | `postgresql://USER:PASSWORD@HOST:5432/DB`. With the bundled `postiz-postgres` service, keep it in sync with that service's `POSTGRES_USER`, `POSTGRES_PASSWORD` and `POSTGRES_DB`. |
| `REDIS_URL` | `redis://postiz-redis:6379` for the bundled Redis. Must start with `redis://`. |
| `BACKEND_INTERNAL_URL` | URL the frontend uses to reach the API inside the container. Leave `http://localhost:3000`. |
| `TEMPORAL_ADDRESS` | `temporal:7233` for the bundled Temporal service. |
| `IS_GENERAL` | Leave `'true'` for self-hosting. |
| `DISABLE_REGISTRATION` | `'true'` allows sign-up only while no organization exists (your first account); after that, new users must be invited. Generic OAuth sign-ins are still allowed. `'false'` allows open sign-ups. |

The app checks `JWT_SECRET`, `MAIN_URL`, `FRONTEND_URL`, `NEXT_PUBLIC_BACKEND_URL`, `BACKEND_INTERNAL_URL`, `DATABASE_URL`, `REDIS_URL` and `STORAGE_PROVIDER` on startup and logs any that are missing or invalid.

### HTTPS and `redirectmeto.com`

Some platforms (Threads, TikTok, Slack, Apple) only accept `https://` callback URLs. When `FRONTEND_URL` is plain `http://`, Postiz prefixes those callbacks with `https://redirectmeto.com/` so local testing still works. For production, serve Postiz over HTTPS (behind a reverse proxy such as Caddy, Nginx or Traefik) and register the plain `https://` callbacks.

## Storage

| Variable | What to set |
| --- | --- |
| `STORAGE_PROVIDER` | `local` (files go to the `postiz-uploads` volume) or `cloudflare` (Cloudflare R2). |
| `UPLOAD_DIRECTORY` | Folder inside the container for `local` storage. Leave `/uploads`. |
| `NEXT_PUBLIC_UPLOAD_DIRECTORY` | URL path that serves uploaded files. Leave `/uploads`. |

### Cloudflare R2

1. In the [Cloudflare dashboard](https://dash.cloudflare.com), open **R2 Object Storage** and create a bucket. Its name is `CLOUDFLARE_BUCKETNAME`.
2. Your **Account ID** is shown on the R2 overview page: `CLOUDFLARE_ACCOUNT_ID`.
3. Go to **R2 → Manage R2 API Tokens → Create API token**, with **Object Read & Write** on the bucket. Copy the **Access Key ID** (`CLOUDFLARE_ACCESS_KEY`) and **Secret Access Key** (`CLOUDFLARE_SECRET_ACCESS_KEY`). The secret is shown once.
4. In the bucket's **Settings**, enable public access (an `r2.dev` subdomain or a custom domain). That public URL is `CLOUDFLARE_BUCKET_URL`. Social platforms download your media from it, so it must be publicly reachable.
5. Set `CLOUDFLARE_REGION` to `auto` and `STORAGE_PROVIDER` to `cloudflare`.

## Social media channels

Each channel needs an app (or "client") created in that platform's developer portal. The general pattern is the same everywhere:

1. Create an app in the developer portal.
2. Register the **callback / redirect URL** from the table below.
3. Request the listed permissions (scopes).
4. Copy the app's ID and secret into the matching variables.

Callback URLs must match character for character, including `http` vs `https` and the port.

| Channel | Variables | Callback URL to register |
| --- | --- | --- |
| X (Twitter) | `X_API_KEY`, `X_API_SECRET`, `X_URL` | `FRONTEND_URL/integrations/social/x` |
| LinkedIn / LinkedIn Page | `LINKEDIN_CLIENT_ID`, `LINKEDIN_CLIENT_SECRET` | `FRONTEND_URL/integrations/social/linkedin` and `FRONTEND_URL/integrations/social/linkedin-page` |
| Reddit | `REDDIT_CLIENT_ID`, `REDDIT_CLIENT_SECRET` | `FRONTEND_URL/integrations/social/reddit` |
| Facebook | `FACEBOOK_APP_ID`, `FACEBOOK_APP_SECRET` | `FRONTEND_URL/integrations/social/facebook` |
| Instagram (via Facebook) | `FACEBOOK_APP_ID`, `FACEBOOK_APP_SECRET` | `FRONTEND_URL/integrations/social/instagram` |
| Threads | `THREADS_APP_ID`, `THREADS_APP_SECRET` | `FRONTEND_URL/integrations/social/threads` (HTTPS) |
| YouTube | `YOUTUBE_CLIENT_ID`, `YOUTUBE_CLIENT_SECRET` | `FRONTEND_URL/integrations/social/youtube` |
| TikTok | `TIKTOK_CLIENT_ID`, `TIKTOK_CLIENT_SECRET` | `FRONTEND_URL/integrations/social/tiktok` (HTTPS) |
| TikTok Business | `TIKTOK_BUSINESS_CLIENT_ID`, `TIKTOK_BUSINESS_CLIENT_SECRET` | `FRONTEND_URL/integrations/social/tiktok-business/` (HTTPS, **with** trailing slash) |
| Pinterest | `PINTEREST_CLIENT_ID`, `PINTEREST_CLIENT_SECRET` | `FRONTEND_URL/integrations/social/pinterest` |
| Dribbble | `DRIBBBLE_CLIENT_ID`, `DRIBBBLE_CLIENT_SECRET` | `FRONTEND_URL/integrations/social/dribbble` |
| Tumblr | `TUMBLR_CLIENT_ID`, `TUMBLR_CLIENT_SECRET` | `FRONTEND_URL/integrations/social/tumblr` |
| Discord | `DISCORD_CLIENT_ID`, `DISCORD_CLIENT_SECRET`, `DISCORD_BOT_TOKEN_ID` | `FRONTEND_URL/integrations/social/discord` |
| Slack | `SLACK_ID`, `SLACK_SECRET` | `FRONTEND_URL/integrations/social/slack` (HTTPS) |
| Mastodon | `MASTODON_URL`, `MASTODON_CLIENT_ID`, `MASTODON_CLIENT_SECRET` | `FRONTEND_URL/integrations/social/mastodon` |

### X (Twitter)

1. Sign in at the [X Developer Portal](https://developer.x.com/en/portal/dashboard) and create a Project and an App.
2. Under **User authentication settings**, enable **OAuth 1.0a** with **Read and write** permissions. Set the app type to **Web App** and add the callback URL. A website URL is also required.
3. Under **Keys and tokens → Consumer Keys**, copy the **API Key** to `X_API_KEY` and the **API Key Secret** to `X_API_SECRET`.
4. `X_URL` is optional. Set it only if X must call back to a different host than `FRONTEND_URL`; otherwise leave it empty.

Optional: `DISABLE_X_ANALYTICS=true` hides X analytics (useful on the free API tier), `STRIP_LINKS_FROM_X_POSTS=true` removes links from X posts.

### LinkedIn

1. Create an app at [LinkedIn Developers](https://www.linkedin.com/developers/apps). You need a LinkedIn Company Page to associate it with.
2. Under **Products**, add **Share on LinkedIn**, **Sign In with LinkedIn using OpenID Connect** and, to post as a company page, **Advertising API** (approval required).
3. Under **Auth**, add both callback URLs and copy the **Client ID** and **Primary Client Secret**.

Scopes used: `openid`, `profile`, `w_member_social`, `r_basicprofile`, `rw_organization_admin`, `w_organization_social`, `r_organization_social`.

### Reddit

1. Go to [reddit.com/prefs/apps](https://www.reddit.com/prefs/apps) and click **create another app**.
2. Choose type **web app** and enter the callback URL as the **redirect uri**.
3. The string under the app name is `REDDIT_CLIENT_ID`; **secret** is `REDDIT_CLIENT_SECRET`.

### Facebook and Instagram

Both use the same Meta app.

1. Create an app at [Meta for Developers](https://developers.facebook.com/apps) with the **Business** type.
2. Add the **Facebook Login for Business** product and, for Instagram, the **Instagram Graph API** product.
3. Under **Facebook Login → Settings → Valid OAuth Redirect URIs**, add both the Facebook and Instagram callback URLs.
4. Under **App settings → Basic**, copy the **App ID** and **App Secret**.
5. To let accounts other than the app's admins and testers connect, switch the app to **Live** and pass Meta's App Review for the permissions below.

Facebook permissions: `pages_show_list`, `business_management`, `pages_manage_posts`, `pages_manage_engagement`, `pages_read_engagement`, `read_insights`.
Instagram permissions: `instagram_basic`, `pages_show_list`, `pages_read_engagement`, `business_management`, `instagram_content_publish`, `instagram_manage_comments`, `instagram_manage_insights`. The Instagram account must be a Business or Creator account linked to a Facebook Page.

### Threads

1. In [Meta for Developers](https://developers.facebook.com/apps), create an app and choose the **Access the Threads API** use case.
2. In the Threads use case settings, add the callback URL to **Redirect Callback URLs** (and the uninstall/delete callbacks, which can point to the same URL).
3. Add the permissions `threads_basic`, `threads_content_publish`, `threads_manage_replies`, `threads_manage_insights`.
4. Copy the **Threads App ID** and **Threads App Secret** (these are different from the main Meta app ID and secret).
5. While the app is in development, add your Threads account under **App roles → Roles → Threads Testers** and accept the invite in the Threads app (Settings → Account → Website permissions).

### YouTube

1. In the [Google Cloud Console](https://console.cloud.google.com), create a project.
2. Under **APIs & Services → Library**, enable **YouTube Data API v3**, **YouTube Analytics API** and **YouTube Reporting API**.
3. Configure the **OAuth consent screen** (External), add your Google account as a test user while the app is in testing.
4. Under **Credentials → Create credentials → OAuth client ID**, choose **Web application** and add the callback URL under **Authorized redirect URIs**.
5. Copy the **Client ID** and **Client secret**.

### TikTok

1. Create an app at [TikTok for Developers](https://developers.tiktok.com/apps).
2. Add the **Login Kit** and **Content Posting API** products. Enable **Direct Post** in Content Posting API.
3. Register the callback URL under Login Kit's **Redirect URI**, and verify your domain if asked.
4. Request the scopes `user.info.basic`, `user.info.profile`, `user.info.stats`, `video.list`, `video.upload`, `video.publish`.
5. Copy the **Client key** to `TIKTOK_CLIENT_ID` and the **Client secret** to `TIKTOK_CLIENT_SECRET`.

TikTok only accepts HTTPS callbacks and requires app review before accounts other than sandbox testers can connect.

**TikTok Business** works the same way with a separate Business app: register `FRONTEND_URL/integrations/social/tiktok-business/` (the trailing slash is required) and use `TIKTOK_BUSINESS_CLIENT_ID` and `TIKTOK_BUSINESS_CLIENT_SECRET`.

### Pinterest

1. Create an app at [Pinterest Developers](https://developers.pinterest.com/apps/) (requires a Pinterest business account).
2. Add the callback URL under **Redirect URIs**.
3. Request `boards:read`, `boards:write`, `pins:read`, `pins:write`, `user_accounts:read`. New apps start with Trial access; request Standard access to post publicly.
4. Copy the **App ID** and **App secret key**.

### Dribbble

1. Register an application at [dribbble.com/account/applications](https://dribbble.com/account/applications/new).
2. Set the **Callback URL**.
3. Copy the **Client ID** and **Client Secret**.

### Tumblr

1. Register an application at [tumblr.com/oauth/apps](https://www.tumblr.com/oauth/apps).
2. Set the **Default callback URL** and **OAuth2 redirect URLs** to the callback URL.
3. Copy the **OAuth Consumer Key** to `TUMBLR_CLIENT_ID` and the **Secret Key** to `TUMBLR_CLIENT_SECRET`.

### Discord

1. Create an application at the [Discord Developer Portal](https://discord.com/developers/applications).
2. Under **OAuth2**, copy the **Client ID** and **Client Secret**, and add the callback URL under **Redirects**.
3. Under **Bot**, click **Reset Token** and copy the token to `DISCORD_BOT_TOKEN_ID`. Postiz posts to channels as this bot.

### Slack

1. Create an app at [api.slack.com/apps](https://api.slack.com/apps) (**From scratch**).
2. Under **OAuth & Permissions**, add the callback URL to **Redirect URLs** and add these **Bot Token Scopes**: `channels:read`, `chat:write`, `users:read`, `groups:read`, `channels:join`, `chat:write.customize`.
3. Under **Basic Information → App Credentials**, copy the **Client ID** to `SLACK_ID` and **Client Secret** to `SLACK_SECRET`.
4. To install it in workspaces other than your own, enable **Manage Distribution**.

`SLACK_SIGNING_SECRET` is not used by the current code; you can leave it empty.

### Mastodon

1. On your Mastodon instance, go to **Preferences → Development → New application**.
2. Set the **Redirect URI** to the callback URL and select the scopes `write:statuses`, `write:media` and `profile`.
3. Copy the **Client key** to `MASTODON_CLIENT_ID` and the **Client secret** to `MASTODON_CLIENT_SECRET`.
4. Set `MASTODON_URL` to the instance the app was created on (default `https://mastodon.social`).

Users on other instances can still connect through the **Mastodon (custom instance)** channel, which needs no environment variables.

### Channels without environment variables

Bluesky, Telegram (needs `TELEGRAM_TOKEN` and `TELEGRAM_BOT_NAME` from [@BotFather](https://t.me/BotFather) — not in the default compose file, add them if you want it), Lemmy, Nostr, Dev.to, Hashnode, Medium, WordPress, Farcaster and similar channels ask for credentials in the UI when you connect them.

## Sign-in providers

These let users log in to Postiz itself; they are not posting channels.

### GitHub login

1. Create an OAuth App at [github.com/settings/developers](https://github.com/settings/developers).
2. Set **Homepage URL** to `FRONTEND_URL` and **Authorization callback URL** to `FRONTEND_URL/settings`.
3. Copy the **Client ID** to `GITHUB_CLIENT_ID` and generate a **Client secret** for `GITHUB_CLIENT_SECRET`.

### Sign in with Apple

Requires a paid Apple Developer account, in [Certificates, Identifiers & Profiles](https://developer.apple.com/account/resources/identifiers/list).

1. `APPLE_TEAM_ID`: shown at the top right of the developer account (10 characters).
2. Create a **Services ID**, enable **Sign in with Apple**, and add `NEXT_PUBLIC_BACKEND_URL/auth/oauth/apple/redirect` as the return URL (HTTPS only). The Services ID identifier is `APPLE_CLIENT_ID`.
3. Under **Keys**, create a key with **Sign in with Apple** enabled. Its **Key ID** is `APPLE_KEY_ID`. Download the `.p8` file once and put its full contents (including the `BEGIN`/`END` lines) in `APPLE_PRIVATE_KEY`.

### Generic OAuth / OIDC (Authentik, Keycloak, ...)

Uncomment the `OAuth & Authentik Settings` block and set `POSTIZ_GENERIC_OAUTH: 'true'`.

1. In your identity provider, create an OAuth2/OpenID provider and application. Set its redirect URI to `FRONTEND_URL/settings`.
2. Copy the client ID and secret into `POSTIZ_OAUTH_CLIENT_ID` and `POSTIZ_OAUTH_CLIENT_SECRET`.
3. Fill in the authorize, token and userinfo endpoints from the provider's OpenID configuration (`/.well-known/openid-configuration`). The compose file shows the Authentik paths.
4. `NEXT_PUBLIC_POSTIZ_OAUTH_DISPLAY_NAME` and `NEXT_PUBLIC_POSTIZ_OAUTH_LOGO_URL` control the login button's label and icon.

## Newsletter (Beehiiv)

Optional. When `BEEHIIVE_API_KEY` is set, new users are subscribed to your Beehiiv publication.

1. In Beehiiv, go to **Settings → Integrations → API** and create an API key: `BEEHIIVE_API_KEY`.
2. The publication ID (starts with `pub_`) is shown on the same page: `BEEHIIVE_PUBLICATION_ID`.

## AI and design

| Variable | Where to get it |
| --- | --- |
| `OPENAI_API_KEY` | [platform.openai.com/api-keys](https://platform.openai.com/api-keys). Enables AI post writing and image generation. Requires billing on the OpenAI account. |
| `OPENAI_OAUTH_CLIENT_ID` | Only for publishing Postiz as a ChatGPT app/MCP connector: the OAuth client ID that ChatGPT registers with your Postiz install. Leave empty otherwise. |
| `NEXT_PUBLIC_POLOTNO` | API key from [polotno.com/cabinet](https://polotno.com/cabinet). Enables the built-in image design editor. |

## Misc settings

| Variable | What to set |
| --- | --- |
| `NEXT_PUBLIC_DISCORD_SUPPORT` | Optional invite link to your Discord server, shown as a support link. |
| `API_LIMIT` | Public API requests allowed per hour. The compose file sets `30`; if removed, the default is `90`. |
| `NX_ADD_PLUGINS` | Build-time setting. Leave `false`. |

## Payments (Stripe)

Only needed if you want to charge for Postiz subscriptions. Leave empty for a private self-hosted install.

1. In the [Stripe dashboard](https://dashboard.stripe.com/apikeys), copy the **Publishable key** (`STRIPE_PUBLISHABLE_KEY`) and **Secret key** (`STRIPE_SECRET_KEY`).
2. Under **Developers → Webhooks**, add an endpoint at `NEXT_PUBLIC_BACKEND_URL/stripe`. Copy its **Signing secret** (`whsec_...`) to `STRIPE_SIGNING_KEY`.

`FEE_AMOUNT` and `STRIPE_SIGNING_KEY_CONNECT` are not used by the current code.

## Short links (optional)

Uncomment one provider to shorten links in posts automatically.

| Provider | Variables | Where to get them |
| --- | --- | --- |
| Dub | `DUB_TOKEN`, `DUB_API_ENDPOINT`, `DUB_SHORT_LINK_DOMAIN` | [app.dub.co](https://app.dub.co) → Settings → API Keys. Domain is `dub.sh` or your custom domain. |
| Short.io | `SHORT_IO_SECRET_KEY` | [app.short.io](https://app.short.io) → Integrations & API → API → secret key. |
| Kutt | `KUTT_API_KEY`, `KUTT_API_ENDPOINT`, `KUTT_SHORT_LINK_DOMAIN` | Your Kutt instance (or [kutt.it](https://kutt.it)) → Settings → API. |
| LinkDrip | `LINK_DRIP_API_KEY`, `LINK_DRIP_API_ENDPOINT`, `LINK_DRIP_SHORT_LINK_DOMAIN` | LinkDrip dashboard → API settings. |

## Monitoring (Sentry / Spotlight)

`NEXT_PUBLIC_SENTRY_DSN` and `SENTRY_SPOTLIGHT` send errors to the bundled `spotlight` container for local debugging (open `http://localhost:8969`). For real Sentry, set `NEXT_PUBLIC_SENTRY_DSN` to the DSN from your Sentry project (**Settings → Client Keys (DSN)**) and leave `SENTRY_SPOTLIGHT` unset.

## Troubleshooting

- **"redirect_uri mismatch" / "invalid redirect"**: the callback registered with the platform doesn't exactly match `FRONTEND_URL/integrations/social/<channel>`. Check scheme, port and trailing slash.
- **A channel is missing from "Add channel"**: its client ID/secret variables are empty, or the container wasn't recreated after setting them.
- **Media fails to post on some platforms**: the platform must be able to download your media, so the upload URL (local `FRONTEND_URL/uploads` or `CLOUDFLARE_BUCKET_URL`) must be publicly reachable over the internet.
