# Changelog

## [1.6.0] - 2026-10-14

### Added
- **Dynamic upstream DNS configuration** — replaced hardcoded `9.9.9.9` with runtime-configurable resolver.
  - New endpoints: `POST /setupstream` and `POST /setdns` (requires Basic Auth if enabled + CSRF token, validates IPv4 address).
  - Config persisted to LittleFS in `/dns.cfg` across reboots.
  - Current upstream DNS reported in `/stats.json` (`upstream` field).
  - Dashboard UI updated under **Network → UPSTREAM DNS** showing current upstream resolver, text field to set any custom DNS server, and quick preset buttons for Quad9 (`9.9.9.9`), Cloudflare (`1.1.1.1`), and Google (`8.8.8.8`).
  - Upstream DNS status card added to dashboard header cards.
  - Factory reset updated to erase `/dns.cfg` and revert to default Quad9 (`9.9.9.9`).
  - Dashboard ETag bumped to `c3adb-v3` to invalidate client browser caches.

### Files Changed
- `src/main.cpp` — Defined `DEFAULT_UPSTREAM` and mutable `upstreamDns`; added `/dns.cfg` load/save persistence; added `handleSetUpstream()` endpoint; updated UDP and TCP forwarders; updated `/stats.json`, factory reset, and boot logs.
- `src/dashboard_html.h` — Added upstream DNS UI and presets in Network section; added DNS card in header; added `saveUpstream()` and `setUpstreamPreset()` JavaScript handlers; updated factory reset description.
- `README.md` — Documented dashboard upstream DNS configuration under OTA updates section.

### Testing
- Verified: `tools/test_build_blocklist.py` passed (33 passed, 0 failed).
- Verified: HTML/JS structure and element IDs (`updns`, `udns`, `udnsmsg`) parsed cleanly via HTML parser.
- Verified: C++ variable references, route bindings (`/setupstream`, `/setdns`), packet forwarding calls, and LittleFS file operations are consistent.

---

## [1.5.0] - 2026-07-25

### Changed
- **Dashboard UI reorganized into collapsible sections** — cluttered flat layout replaced with 4 grouped `<details>/<summary>` sections for better usability.
  - Blocklist (expanded) — Upload + Remote Auto-Update
  - Network (expanded) — WiFi + DHCP
  - Security (expanded) — Password Change + Auth Toggle
  - Advanced (collapsed) — Firmware OTA + Factory Reset
- Replaced `LOGN` (Serial.println) calls with `LOG` (Serial.printf) for formatted debug output in password change, auth toggle, and factory reset handlers.
- **Firmware OTA help text updated** — removed confusing `.pio/build/c3/firmware.bin` path reference, now links to source repo and tells users to build from source for their board.
- **License compliance and attribution** — updated LICENSE to credit M-Abozaid as original author; added forked-from note and credits section in README.

### Files Changed
- `src/dashboard_html.h` — Reorganized all dashboard sections into collapsible groups; added details/summary elements with proper open/closed defaults; updated firmware OTA help text
- `src/main.cpp` — Fixed LOGN to LOG calls for printf-style formatting
- `LICENSE` — Added M-Abozaid copyright
- `README.md` — Added forked-from note and updated Credits section

### Testing
- Verified: All 7 element IDs present in HTML (upload, uploadstatus, dhcp, remotelist, firmware, factoryreset, resetStatus)
- Verified: 3 sections expanded by default (Blocklist, Network, Security), Advanced collapsed
- Verified: All endpoints functional via curl (setpassword, toggleauth, factoryreset)
- Verified: Firmware OTA help text links to GitHub repo and no longer references PlatformIO build path

---

## [1.4.0] - 2026-07-25

### Changed
- **Auth disabled by default on first boot** — new devices boot with authentication disabled, dashboard accessible immediately without credentials.
  - Random password still generated and stored in `/auth.cfg`
  - User can set their own password and enable auth from the dashboard
  - After factory reset, auth is also disabled (same as first boot)

### Added
- **Password change without current password when auth is disabled** — when auth is off, the "Current password" field is optional for setting a new password.
  - This is consistent with the security model: no auth = no security to protect
  - When auth is enabled, current password is still required (existing behavior)

### Files Changed
- `src/main.cpp` — Modified `loadAuth()` to write `"0"` on first boot, modified `handleSetPassword()` to skip current password check when `!authEnabled`
- `src/dashboard_html.h` — Updated password field placeholder text to indicate "not needed if auth disabled"

### Testing
- Verified: Auth disabled after factory reset
- Verified: Dashboard accessible without credentials
- Verified: Password change works without current password when auth disabled
- Verified: Password change requires current password when auth enabled
- Verified: New password works after change

---

## [1.3.0] - 2026-07-25

### Added
- **Factory reset feature** — users can erase all config and reboot into setup mode from the dashboard.
  - New endpoint: `POST /factoryreset`
  - Parameters: `confirm` (must be "RESET"), `token` (CSRF)
  - Erases: `/auth.cfg`, `/wifi.cfg`, `/custom.txt`, `/banned.txt`, `/update.cfg`, `/dhcp.cfg`, `/blocklist.bin`, `/blocklist.new`
  - Device reboots into captive portal mode after reset
  - New UI section "FACTORY RESET" at bottom of dashboard with confirmation input

### Security
- Factory reset requires Basic Auth + CSRF token
- Requires typing "RESET" as confirmation to prevent accidental clicks
- Red warning text explains what will be erased

### Files Changed
- `src/main.cpp` — Added `handleFactoryReset()` function and `/factoryreset` endpoint registration
- `src/dashboard_html.h` — Added factory reset section with confirmation input and `factoryReset()` JavaScript function

### Testing
- Verified: Wrong confirmation word rejected with "type RESET to confirm"
- Verified: Correct confirmation triggers reset and reboot
- Verified: New random password generated after reset
- Verified: Custom domains, clients, bans cleared after reset
- Note: Blocklist data may survive (regeneration requires Python toolchain)

---

## [1.2.0] - 2026-07-25

### Added
- **Dashboard authentication toggle** — users can enable/disable webUI authentication from the dashboard.
  - New endpoint: `POST /toggleauth`
  - Parameters: `token` (CSRF)
  - Auth state persisted to `/auth.cfg` line 3 in LittleFS
  - When auth is disabled, all endpoints accessible without credentials
  - New UI toggle button "Enable / Disable Auth" in DASHBOARD PASSWORD section
  - Red warning text: "WARNING: Anyone on your network can flash firmware and change settings when auth is disabled"

### Security
- Toggle requires Basic Auth + CSRF token (when auth is enabled)
- When auth is disabled, the toggle endpoint is unprotected (anyone can re-enable)
- Stats JSON endpoint returns `auth` field showing current state

### Files Changed
- `src/main.cpp` — Added `authEnabled` global, `saveAuthCfg()`, `handleToggleAuth()` function, `/toggleauth` endpoint
- `src/dashboard_html.h` — Added auth toggle button, `authst` status display, `toggleAuth()` JavaScript function

### Testing
- Verified: Auth enabled by default after flash
- Verified: Toggle disables auth — all endpoints accessible without credentials
- Verified: Toggle re-enables auth — credentials required again
- Verified: Stats JSON includes `auth` field

---

## [1.1.0] - 2026-07-25

### Added
- **Dashboard password change feature** — users can now change the web UI password from the dashboard without reflashing.
  - New endpoint: `POST /setpassword`
  - Parameters: `current`, `newpass`, `confirm`, `token` (CSRF)
  - Validation:
    - Current password must match
    - New password must be at least 6 characters
    - New password and confirmation must match
  - Password is persisted to `/auth.cfg` in LittleFS
  - New UI section "DASHBOARD PASSWORD" added to dashboard between WiFi and Firmware sections

### Security
- Password change requires Basic Auth + CSRF token
- Current password verification prevents unauthorized changes
- Minimum password length (6 chars) enforced

### Files Changed
- `src/main.cpp` — Added `handleSetPassword()` function and `/setpassword` endpoint registration
- `src/dashboard_html.h` — Added password change form and `savePw()` JavaScript function

### Testing
- Verified: New password works after change
- Verified: Old password is rejected after change
- Verified: Wrong current password is rejected
- Verified: Short passwords (< 6 chars) are rejected
- Verified: Password mismatch is rejected

---

## [1.0.0] - Original Release

### Features
- DNS sinkhole with ~140k domains (40-bit FNV-1a hashes in flash)
- Bloom filter pre-filter (128 KB RAM)
- Async DNS forwarding (32 concurrent queries)
- TCP DNS support on port 53
- AAAA sinkhole (blocked IPv6 queries answer `::`)
- Web dashboard with Basic Auth + CSRF token
- WiFiManager captive portal for first-boot WiFi setup
- Optional DHCP server
- OTA firmware and blocklist updates
