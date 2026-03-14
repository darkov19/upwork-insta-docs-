# Product Brief: Stealth Web Scraper & Activity Monitoring Platform

## Overview

This project aims to build a **stealth web-based application** that continuously scrapes public data from Instagram (and potentially Facebook) user profiles, detects changes (e.g., follows/unfollows), and delivers **real-time alerts** to end users. The application must be **bot-resistant**, **privacy-focused**, and **launch-ready** within a short development cycle.

---

## Goals

- ✅ Build a **high-frequency data ingestion system** that avoids detection
- ✅ Track **"followers" and "following" changes** with snapshot diffing
- ✅ Provide **real-time alerts** on activity
- ✅ Enable **anonymous story viewing** without sending view receipts
- ✅ Design the platform to be scalable and stealthy

---

## Core Features

### 1. Public Data Ingestion Engine

> Scrape profile data like followers/following from public Instagram pages using human-like automation.

**Technical Requirements:**

- Use **Playwright** or **Puppeteer** for headless browsing
- Implement **residential/mobile proxy rotation** (Bright Data, Oxylabs)
- Detect and extract structured data (JSON-LD, HTML scraping)
- Handle anti-bot mechanisms (timeouts, captchas, fingerprinting)

---

### 2. State-Based Difference Engine

> Detect follower/following changes using snapshot comparisons.

**Technical Requirements:**

- Store profile snapshots (e.g., user ID + follower/following list) in **MongoDB** or **PostgreSQL**
- Use a **diffing algorithm** to detect adds/removes
- Support snapshot intervals of **5–15 minutes**
- Optionally reconstruct a **timeline of activity**

---

### 3. Real-Time Alerting Pipeline

> Push alerts to users based on follower activity.

**Technical Requirements:**

- Use **Celery (Python)** or **BullMQ (Node.js)** for background task management
- Trigger events when changes are detected
- Integrate **Firebase Cloud Messaging (FCM)** or **Apple Push Notification service (APNs)**
- Queue management for **thousands of users** concurrently

---

### 4. Anonymous Story Viewing

> Allow users to watch Instagram stories without triggering a "seen" status.

**Technical Requirements:**

- Backend fetches `.mp4` or `.jpg` files directly
- Use **server-side proxying** to avoid client attribution
- Implement **sessionless scraping** (no logged-in user)
- Cache content via **Cloudflare or Varnish** for multi-user delivery

---

## Nice-to-Have (Bonus Features)

- Instagram/Facebook API integration (if rate-limits allow)
- User dashboard with visual timeline of follows/unfollows
- Optional stealth browser fingerprinting tools
- AI Agent that suggests users to monitor based on interaction trends

---

## Target Users

- Social media analysts
- Influencers and brand managers
- Privacy-focused users
- Competitive research teams

---

## Constraints

- Must operate within **Instagram’s public data boundaries** (no credential scraping)
- Must avoid rate-limiting and bans
- Must be **launch-ready in days**, not weeks

---

## Tech Stack (Suggested)

| Layer              | Tech                        |
| ------------------ | --------------------------- |
| Frontend           | React / Next.js             |
| Backend            | Node.js (or Python FastAPI) |
| Scraping           | Puppeteer / Playwright      |
| Storage            | MongoDB or PostgreSQL       |
| Queueing           | Celery + Redis / BullMQ     |
| Proxy              | Bright Data / Oxylabs       |
| Push Notifications | FCM / APNs                  |
| Caching            | Cloudflare, Varnish         |

---

## Deliverables

1. Web scraper (bot-evading) with proxy rotation
2. Snapshot diffing engine with history logging
3. Real-time push notification system
4. Anonymous story fetcher with CDN caching
5. Launch-ready dashboard for MVP

---

## Timeline

- **Day 1–2**: Finalize tech stack & set up proxy & scraping layer
- **Day 3–5**: Implement diffing engine + task scheduling
- **Day 6–7**: Set up push notification pipeline & test
- **Day 8–9**: Anonymous story fetcher + caching
- **Day 10**: Polish UI + prepare for launch

---

## Budget

- 💰 Fixed price: $200 (client open to higher for top-tier speed and delivery)
- 💼 Contract-to-hire potential

---

## Contact

Please send:

- Code samples (especially scrapers or stealth tools)
- Similar projects (links, repos, or videos)
- Your proposed approach (brief outline is enough)
