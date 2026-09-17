# RestroDesk — Shop Installation Guide

Everything needed to run this system (POS, billing, table QR ordering,
kitchen display, reports) on one machine inside a shop, reachable by every
staff device and customer phone on that shop's own WiFi — no internet
dependency once it's running.

One Docker install = one restaurant business (owner, branches, staff,
menu, tables all belong to it). A business with several branches still
uses a single install; a second, unrelated shop needs its own separate
install (its own machine, its own `.env`, its own database).

## What you need before you start

- One machine to act as the server: a PC, mini-PC, or NUC, connected to
  the shop's router **by Ethernet cable** (WiFi for the server itself is
  possible but less reliable for something running all day).
- [Docker Engine and the Docker Compose plugin](https://docs.docker.com/engine/install/)
  installed on that machine.
- Admin access to the shop's WiFi router, to reserve a fixed IP for the
  server (every printed table QR code encodes this IP — it cannot change
  later without reprinting every QR code and rebuilding the app).

## First-time setup

**1. Reserve a static IP for the server**
In the router's admin page, find the server machine (by its MAC address)
and set a DHCP reservation, e.g. `192.168.1.50`. Confirm it with:
```
ip addr show   # Linux
ipconfig       # Windows
```

**2. Get the code onto the server machine**
Copy or clone this repository onto the server machine.

**3. Configure the install**
```
cp .env.example .env
```
Edit `.env` and set, at minimum:
- `SERVER_LAN_IP` — the static IP from step 1
- `POSTGRES_PASSWORD` — a real password
- `JWT_SECRET` — a long random string (`openssl rand -hex 32` generates one).
  The backend refuses to start if this is left as the placeholder or is
  under 32 characters.
- `SUPER_ADMIN_EMAIL` / `SUPER_ADMIN_PASSWORD` — the first login for this
  install, used to create the shop's owner account from `/platform`.
  The password needs at least 8 characters with a letter and a number, or
  startup fails the same way.

Leave `COOKIE_SECURE` and `DB_SSL` at their defaults (`false`) unless
you're doing something non-standard (HTTPS reverse proxy, external managed
database) — see the comments above each in `.env.example`.

**4. Build and start everything**
```
docker compose up -d --build
```
This starts Postgres, runs database migrations, seeds the platform role
and Starter plan, creates the super admin account, and starts the backend
(port 5000) and frontend (port 3000). First run takes a few minutes while
images build; watch progress with:
```
docker compose logs -f
```

**5. Confirm it's reachable on the LAN**
From a *different* device on the same WiFi (a phone works well):
```
http://<SERVER_LAN_IP>:3000
```
If it doesn't load, see Troubleshooting below before going further.

**6. Create the shop's account**
Log in at `/platform` with the `SUPER_ADMIN_EMAIL`/`SUPER_ADMIN_PASSWORD`
from step 3, and create the vendor (the restaurant business), its first
branch, and its owner login. On the create-vendor form, check the
**Country** field — it defaults to India and controls how tax is shown
on bills (India splits it into CGST/SGST; every other country shows a
single "Tax" line), so set it explicitly if the shop is outside India.
Hand the owner login to whoever runs the shop day-to-day — they manage
staff, menu, tables, and branches from there.

**7. Set up tables and print QR codes**
Only after confirming step 5 works: go to **Tables**, add each table, and
use the QR icon on each table card to download or print its code. A
customer scanning it can browse the menu and order straight to that
table.

## Day-to-day operation

| Task | Command |
|---|---|
| Start (after a reboot, if not auto-started) | `docker compose up -d` |
| Stop | `docker compose down` |
| Check everything is actually healthy | `docker compose ps` |
| View live logs | `docker compose logs -f backend` (or `frontend`) |
| Restart just one service | `docker compose restart backend` |
| Back up the database | `docker compose exec postgres pg_dump -U billing_user restaurant_billing > backup.sql` |

Every service has a real health check (Postgres via `pg_isready`, backend
via its `/health` endpoint, frontend via its own HTTP response), so
`docker compose ps` shows `healthy`/`unhealthy` rather than just
"container is running" — and the frontend won't start serving traffic
until the backend is actually ready, not just until its container exists.

Docker's `restart: unless-stopped` policy (already set for every service)
brings everything back up automatically after a power cut or reboot, as
long as Docker itself is set to start on boot:
```
sudo systemctl enable docker
```

**Application logs beyond `docker compose logs`**: the backend also
writes structured logs to `restaurant-billing-backend/logs/` on the host
(bind-mounted, so they survive container rebuilds) — `combined.log` for
every request and business event (order placed, bill generated, staff
changes, etc.) and `error.log` for genuine server errors only. Each file
is capped at 10MB with 5 rotated backups kept, so they won't grow
unbounded on a shop's disk.

## Updating to a newer version later

```
git pull
docker compose up -d --build
```
Migrations run automatically on every backend start and are safe to
re-run.

## Troubleshooting

**Can't reach `http://<SERVER_LAN_IP>:3000` from another device**
- Confirm the server and the other device are on the same WiFi network.
- Check the server's firewall allows inbound TCP 3000 and 5000
  (`sudo ufw allow 3000,5000/tcp` on Ubuntu).
- Re-check the IP with `ip addr show` — it must match `SERVER_LAN_IP` in
  `.env` exactly.

**Changed `SERVER_LAN_IP` and now nothing works**
`NEXT_PUBLIC_API_BASE_URL` is baked into the frontend at build time, not
read at startup. After changing `SERVER_LAN_IP` in `.env`:
```
docker compose up -d --build frontend
```
Then reprint every table's QR code, since they encoded the old IP.

**Bill/invoice PDFs fail to generate**
Check the backend logs (`docker compose logs backend`) for a Chromium
launch error. Rebuild the backend image (`docker compose up -d --build
backend`) — this reinstalls Chromium from scratch.

**Backend container exits immediately on startup**
Check `docker compose logs backend` for one of two deliberate startup
guards: `JWT_SECRET` is missing/too short/still the placeholder, or
`SUPER_ADMIN_PASSWORD` doesn't meet the minimum strength (8+ characters,
a letter and a number). Fix the value in `.env` and
`docker compose up -d --build backend`.

**"Too many requests" errors from the app**
The API rate-limits itself per device — generous for normal use (a few
hundred requests per 15 minutes per device, tighter specifically on
login and on the anonymous QR-ordering endpoints) — as brute-force and
abuse protection. A staff member or customer legitimately hitting it
during normal use is very unlikely; it clears on its own after the
window passes (a few minutes), or restart the backend
(`docker compose restart backend`) to reset it immediately.

**Forgot the super admin password**
Edit `SUPER_ADMIN_PASSWORD` in `.env`, then `docker compose restart
backend` — the entrypoint only *creates* the super admin if the email
doesn't already exist, so also change `SUPER_ADMIN_EMAIL` to a new
address to get a fresh login, or reset the password directly in the
database.

**Starting over completely**
`docker compose down -v` deletes the database and uploaded files along
with the containers — only do this if you genuinely want to wipe the
install and start fresh.
