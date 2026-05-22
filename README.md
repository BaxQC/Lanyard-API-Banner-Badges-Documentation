<div align="center">

# 🎖️ Discord Assets API

### A blazing-fast Go REST API for Discord user badges and banners
*Inspired by [Lanyard](https://github.com/phineas/lanyard)*

<br/>

[![Go Version](https://img.shields.io/badge/Go-1.21+-00acd7?style=for-the-badge&logo=go&logoColor=white)](https://golang.org)
[![License](https://img.shields.io/badge/License-MIT-5865F2?style=for-the-badge)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Active-3ba55c?style=for-the-badge)]()
[![No Dependencies](https://img.shields.io/badge/Dependencies-Zero-faa61a?style=for-the-badge)]()

[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat-square)](CONTRIBUTING.md)
[![Go Report Card](https://goreportcard.com/badge/github.com/yourusername/discord-assets-api?style=flat-square)](https://goreportcard.com/report/github.com/yourusername/discord-assets-api)
[![GitHub Stars](https://img.shields.io/github/stars/yourusername/discord-assets-api?style=flat-square&color=yellow)](https://github.com/yourusername/discord-assets-api/stargazers)

</div>

---

## 📖 What is this?

**Discord Assets API** is a lightweight, self-hostable REST API written in pure Go (zero external dependencies) that exposes Discord user **badges** and **banners** through clean, simple endpoints. Think of it like [Lanyard](https://github.com/phineas/lanyard) — but focused purely on profile assets.

- 🏅 Fetch all badges for any Discord user by their ID
- 🖼️ Proxy banners at any supported resolution
- 📦 Single combined `/profile` endpoint for both at once
- ⚡ Pure `net/http` — no frameworks, no bloat
- 🔓 No Discord bot token or API key required

Data is sourced from [`dcdn.dstn.to`](https://dcdn.dstn.to).

---

## 🚀 Quick Start

```bash
# Clone the repo
git clone https://github.com/yourusername/discord-assets-api
cd discord-assets-api

# Run it
go run main.go

# Server is live at http://localhost:8080
```

> **Requires Go 1.21+** — no other dependencies needed.

---

## 📡 API Reference

### Base URL

```
http://localhost:8080
```

---

### `GET /badges/:id`

Returns all Discord profile badges for a given user.

| Parameter | Type   | Required | Description              |
|-----------|--------|----------|--------------------------|
| `id`      | string | ✅ Yes   | Discord user snowflake ID |

**Example Request**
```http
GET /badges/1409006135087988767
```

**Example Response**
```json
{
  "user_id": "1409006135087988767",
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

---

### `GET /banner/:id`

Proxies the user's Discord banner image directly. Returns the raw image.

| Parameter | Type   | Required | Description                                          |
|-----------|--------|----------|------------------------------------------------------|
| `id`      | string | ✅ Yes   | Discord user snowflake ID                             |
| `size`    | number | ❌ No    | `128`, `256`, `512` *(default)*, `1024`, `2048`      |

**Example Request**
```http
GET /banner/1409006135087988767?size=1024
```

**Response**

Returns `image/png` or `image/gif` directly. Use in an `<img>` tag or download link.

```html
<img src="http://localhost:8080/banner/1409006135087988767?size=512" />
```

**Error Responses**

| Status | Meaning                     |
|--------|-----------------------------|
| `400`  | Missing/invalid ID or size  |
| `404`  | User has no banner set      |
| `502`  | Upstream fetch failed       |

---

### `GET /profile/:id`

Convenience endpoint — returns **badges + banner URL** in a single request.

| Parameter | Type   | Required | Description                                     |
|-----------|--------|----------|-------------------------------------------------|
| `id`      | string | ✅ Yes   | Discord user snowflake ID                        |
| `size`    | number | ❌ No    | Banner size (same options as `/banner`)          |

**Example Request**
```http
GET /profile/1409006135087988767?size=512
```

**Example Response**
```json
{
  "user_id": "1409006135087988767",
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

## 🛠️ Full Source Code

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"strings"
)

// Badge represents a single Discord profile badge.
type Badge struct {
	ID          string `json:"id"`
	Description string `json:"description"`
	Icon        string `json:"icon"`
}

// ProfileResponse is the upstream dstn.to profile JSON shape (partial).
type ProfileResponse struct {
	BadgeEntries []struct {
		ID          string `json:"id"`
		Description string `json:"description"`
		Icon        string `json:"icon"`
	} `json:"badge_entries"`
}

// BadgesReply is what we return to callers of /badges/:id.
type BadgesReply struct {
	UserID string  `json:"user_id"`
	Badges []Badge `json:"badges"`
}

// ProfileReply is the combined response for /profile/:id.
type ProfileReply struct {
	UserID    string  `json:"user_id"`
	BannerURL string  `json:"banner_url"`
	Badges    []Badge `json:"badges"`
}

const (
	profileBase = "https://dcdn.dstn.to/profile/"
	bannerBase  = "https://dcdn.dstn.to/banners/"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/badges/", handleBadges)
	mux.HandleFunc("/banner/", handleBanner)
	mux.HandleFunc("/profile/", handleProfile)

	log.Println("Listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}

// extractID pulls the user ID after the route prefix, stripping query strings.
func extractID(prefix, path string) string {
	id := strings.TrimPrefix(path, prefix)
	id = strings.Split(id, "?")[0]
	return strings.TrimSpace(id)
}

// fetchBadges calls the upstream dstn.to API and parses badge entries.
func fetchBadges(userID string) ([]Badge, error) {
	resp, err := http.Get(profileBase + userID)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	if resp.StatusCode != 200 {
		return nil, fmt.Errorf("upstream returned %d", resp.StatusCode)
	}

	var pr ProfileResponse
	if err := json.NewDecoder(resp.Body).Decode(&pr); err != nil {
		return nil, err
	}

	badges := make([]Badge, 0, len(pr.BadgeEntries))
	for _, e := range pr.BadgeEntries {
		badges = append(badges, Badge{
			ID:          e.ID,
			Description: e.Description,
			Icon:        e.Icon,
		})
	}
	return badges, nil
}

// handleBadges — GET /badges/:id
func handleBadges(w http.ResponseWriter, r *http.Request) {
	userID := extractID("/badges/", r.URL.Path)
	if userID == "" {
		http.Error(w, "missing user ID", http.StatusBadRequest)
		return
	}

	badges, err := fetchBadges(userID)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadGateway)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(BadgesReply{UserID: userID, Badges: badges})
}

// handleBanner — GET /banner/:id?size=512
func handleBanner(w http.ResponseWriter, r *http.Request) {
	userID := extractID("/banner/", r.URL.Path)
	if userID == "" {
		http.Error(w, "missing user ID", http.StatusBadRequest)
		return
	}

	size := r.URL.Query().Get("size")
	if size == "" {
		size = "512"
	}

	allowed := map[string]bool{"128": true, "256": true, "512": true, "1024": true, "2048": true}
	if !allowed[size] {
		http.Error(w, "invalid size; use 128, 256, 512, 1024, or 2048", http.StatusBadRequest)
		return
	}

	upstream := fmt.Sprintf("%s%s?size=%s", bannerBase, userID, size)
	resp, err := http.Get(upstream)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadGateway)
		return
	}
	defer resp.Body.Close()

	if resp.StatusCode == 404 {
		http.Error(w, "user has no banner", http.StatusNotFound)
		return
	}
	if resp.StatusCode != 200 {
		http.Error(w, fmt.Sprintf("upstream returned %d", resp.StatusCode), http.StatusBadGateway)
		return
	}

	w.Header().Set("Content-Type", resp.Header.Get("Content-Type"))
	w.Header().Set("Cache-Control", "public, max-age=3600")
	io.Copy(w, resp.Body)
}

// handleProfile — GET /profile/:id?size=512
func handleProfile(w http.ResponseWriter, r *http.Request) {
	userID := extractID("/profile/", r.URL.Path)
	if userID == "" {
		http.Error(w, "missing user ID", http.StatusBadRequest)
		return
	}

	size := r.URL.Query().Get("size")
	if size == "" {
		size = "512"
	}

	badges, err := fetchBadges(userID)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadGateway)
		return
	}

	bannerURL := fmt.Sprintf("%s%s?size=%s", bannerBase, userID, size)

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(ProfileReply{
		UserID:    userID,
		BannerURL: bannerURL,
		Badges:    badges,
	})
}
```

---

## 🐳 Docker

```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY main.go .
RUN go build -o server main.go

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

```bash
docker build -t discord-assets-api .
docker run -p 8080:8080 discord-assets-api
```

---

## 🔗 Data Sources

| Endpoint          | Source URL                                  |
|-------------------|---------------------------------------------|
| Badges            | `https://dcdn.dstn.to/profile/:id`          |
| Banner            | `https://dcdn.dstn.to/banners/:id?size=:size` |

> This API is a proxy/wrapper around publicly accessible CDN endpoints. It does **not** use the Discord API directly and requires no bot token.

---

## 🤝 Credits & Inspiration

- [**Lanyard**](https://github.com/phineas/lanyard) by [@phineas](https://github.com/phineas) — the original Discord presence API that inspired this project
- [`dcdn.dstn.to`](https://dcdn.dstn.to) — the upstream CDN powering badge and banner data

---

## 📄 License

MIT © [yourusername](https://github.com/yourusername)

---

<div align="center">

⭐ **Star this repo if it helped you!** ⭐

</div>
