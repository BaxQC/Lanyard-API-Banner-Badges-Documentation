<div align="center">

<img src="https://storage.googleapis.com/lanyard/static/lanyardtemplogo.png" width="60" alt="Lanyard logo" />

# lanyard-assets

**Discord badge & banner endpoints as a Lanyard extension**

[![Built on Lanyard](https://img.shields.io/badge/Built%20on-Lanyard-5865F2?style=flat-square&logo=discord&logoColor=white)](https://github.com/Phineas/lanyard)
[![dcdn.dstn.to](https://img.shields.io/badge/Powered%20by-dcdn.dstn.to-7289da?style=flat-square)](https://dcdn.dstn.to)
[![License: MIT](https://img.shields.io/badge/License-MIT-3ba55c?style=flat-square)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-faa61a?style=flat-square)](https://github.com/baxqc/lanyard-assets/pulls)
[![Self-Hostable](https://img.shields.io/badge/Self--Hostable-yes-eb459e?style=flat-square)]()

</div>

---

> **This project works alongside [Phineas/lanyard](https://github.com/Phineas/lanyard).**
> Lanyard exposes your Discord presence. This adds two things Lanyard doesn't cover: **profile badges** and **banners** — sourced from [`dcdn.dstn.to`](https://dcdn.dstn.to).

---

## ✨ What it adds

| Endpoint | What it returns |
|---|---|
| `/badges/:user_id` | All Discord profile badges for a user |
| `/banner/:user_id` | User's banner image (proxied, any size) |
| `/profile/:user_id` | Badges + banner URL in one request |

Everything Lanyard already provides (presence, Spotify, activities, KV store) stays the same at `api.lanyard.rest`. This sits alongside it.

---

## 🚀 Quick Start

### Step 1 — Join the Lanyard Discord

Lanyard monitors users in [its Discord server](https://discord.gg/lanyard). You (or your users) must be in it for presence data to work.

### Step 2 — Pull your Lanyard data

```
GET https://api.lanyard.rest/v1/users/:user_id
```

```json
{
  "success": true,
  "data": {
    "discord_user": {
      "id": "1481422459990442096",
      "username": "yourname",
      "avatar": "abc123",
      "public_flags": 4194432
    },
    "discord_status": "online",
    "activities": [],
    "listening_to_spotify": false
  }
}
```

### Step 3 — Fetch badges and banner

```
GET https://your-deployment.com/badges/1481422459990442096
GET https://your-deployment.com/banner/1481422459990442096?size=512
```

No bot token. No API key. Just a user ID.

---

## 📡 Endpoints

### `GET /badges/:user_id`

Returns all Discord profile badges for the given user.

**Parameters**

| Name | In | Required | Description |
|---|---|---|---|
| `user_id` | path | ✅ | Discord snowflake ID |

**Response `200`**

```json
{
  "user_id": "1481422459990442096",
  "badges": [
    {
      "id": "active_developer",
      "description": "Active Developer",
      "icon": "https://cdn.discordapp.com/badge-icons/6bdc42827a38498929a4920da12695d9.png"
    },
    {
      "id": "verified_bot_developer",
      "description": "Early Verified Bot Developer",
      "icon": "https://cdn.discordapp.com/badge-icons/6df5892791306f35ec167945a8d002cc.png"
    }
  ]
}
```

Badge data is sourced from:
```
https://dcdn.dstn.to/profile/:user_id
```

---

### `GET /banner/:user_id`

Proxies the user's Discord banner image directly. The raw image is returned — plug it straight into an `<img>` tag.

**Parameters**

| Name | In | Required | Description |
|---|---|---|---|
| `user_id` | path | ✅ | Discord snowflake ID |
| `size` | query | ❌ | `128` `256` `512` `1024` `2048` — defaults to `512` |

**Example**

```
GET /banner/1481422459990442096?size=1024
```

```html
<!-- Use directly in HTML -->
<img src="https://your-deployment.com/banner/1481422459990442096?size=512" />
```

Banner data is sourced from:
```
https://dcdn.dstn.to/banners/:user_id?size=:size
```

**Error responses**

| Code | Reason |
|---|---|
| `400` | Missing or invalid user ID / unsupported size |
| `404` | User has no banner set |
| `502` | Upstream CDN unreachable |

---

### `GET /profile/:user_id`

Combined endpoint — returns badges and the banner URL together so you only need one request to render a full profile card.

**Parameters**

| Name | In | Required | Description |
|---|---|---|---|
| `user_id` | path | ✅ | Discord snowflake ID |
| `size` | query | ❌ | Banner size — same options as `/banner` |

**Response `200`**

```json
{
  "user_id": "1481422459990442096",
  "banner_url": "https://dcdn.dstn.to/banners/1409006135087988767?size=512",
  "badges": [
    {
      "id": "active_developer",
      "description": "Active Developer",
      "icon": "https://cdn.discordapp.com/badge-icons/6bdc42827a38498929a4920da12695d9.png"
    }
  ]
}
```

---

## 🔗 Combining with Lanyard

A common pattern is to call Lanyard for presence and this API for assets, then merge them client-side:

```js
const [lanyard, assets] = await Promise.all([
  fetch("https://api.lanyard.rest/v1/users/1409006135087988767").then(r => r.json()),
  fetch("https://your-deployment.com/profile/1409006135087988767").then(r => r.json()),
]);

const profile = {
  ...lanyard.data,
  badges: assets.badges,
  banner_url: assets.banner_url,
};
```

---

## 🛠️ Self-Hosting

> See [Phineas/lanyard](https://github.com/Phineas/lanyard) for self-hosting Lanyard itself. The steps below are for this extension only.

```bash
git clone https://github.com/yourusername/lanyard-assets
cd lanyard-assets
cp .env.example .env
```

```bash
# .env
PORT=8080
```

Then run with your preferred stack. Example with Docker:

```bash
docker build -t lanyard-assets .
docker run -p 8080:8080 lanyard-assets
```

---

## 🔗 Data Sources

| Data | CDN URL |
|---|---|
| Badges | `https://dcdn.dstn.to/profile/:user_id` |
| Banner | `https://dcdn.dstn.to/banners/:user_id?size=:size` |

---

## 🤝 Credits

- [**Phineas/lanyard**](https://github.com/Phineas/lanyard) — the Discord presence API this builds on top of
- [**dcdn.dstn.to**](https://dcdn.dstn.to) — the CDN powering badge and banner data

---

## 📄 License

MIT © [yourusername](https://github.com/baxqc)

---

<div align="center">

⭐ If this helped you, consider starring [Phineas/lanyard](https://github.com/Phineas/lanyard) too!

</div>
