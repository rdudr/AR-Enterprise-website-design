# AR Energy Services — Platform Rules

Status: **rules only, nothing built yet.** This file is the single source of truth for how the
AR Enterprise platform is split across the domain, how data moves between the tools, how things are
named, and how the private side is secured. Read it before writing any code for this platform, and
update it (in the same commit) whenever a decision here changes.

Written: 15 Oct 2026.

---

## 1. The big picture

The platform has three parts that share one set of data:

1. **Public website** — for clients and visitors. Shows who we are, our services, and the *total
   impact* of our audits and the *industrial clusters* we work in. It never shows a client's name.
2. **Ops workspace** (private, login required) — where the team collects field data, processes it,
   generates reports, and manages users, clients and audits.
3. **Data server** — one API and one file store that every tool reads from and writes to. A reading
   taken in a field app is stored once and is then used by processing, reports and the public
   impact numbers. Nothing is re-typed between tools.

```
 Field apps (phone, offline)  ──sync──►  Data server (API + files + DB)  ◄──  Ops workspace (web)
                                                │
                                                └── publishes anonymised totals ──►  Public website
```

---

## 2. Domain and server plan

Domain: **arenergyservices.in** (bought on Hostinger, GitHub already connected).

| Address | What it is | Who can see it | Needs a server? |
|---|---|---|---|
| `arenergyservices.in` (and `www.`) | Public website + public Impact page | Everyone | No — static files are enough |
| `ops.arenergyservices.in` | Ops workspace: data collection, processing, Report Studio, management | Logged-in team only | Yes (web app) |
| `api.arenergyservices.in` | Data server: API, database, uploaded files | Only the apps, never browsed directly | Yes (Node + database + disk) |

Rules:

- The **public site must keep working on static hosting.** It reads a published, anonymised data
  file (`impact.json`) and never calls the private API with a login.
- `ops` and `api` are the only parts that need a real server. Until the server is bought they stay
  undeployed. Do **not** put private tools on the free trial hosting.
- Hostinger's free deploy access is temporary (about one month — check the exact end date in
  hPanel and write it here: `TRIAL ENDS: ____`). Before it ends, choose one of:
  - keep the public site on Hostinger web hosting (GitHub auto-deploy), or move it to Vercel; and
  - buy a server for `ops` + `api` (Hostinger VPS or a Node.js-capable plan) with a PostgreSQL
    database and enough disk for photos and Excel files.
- HTTPS everywhere. DNS for all three addresses stays in Hostinger.
- Each part deploys from GitHub. Pushing to `main` deploys; work in progress goes on a branch.

---

## 3. Naming rules (new names, not the old project names)

The old KISEM / IIT Gandhinagar project names (FOX / fox-kisem, Jet EFF, A-CMP, Pump-Up, Zero Degree,
Thermo-X, AI-PQA, PostMan, KISEM) **must not appear** in this platform's code, URLs, screens,
reports, file names or repository names. They were built for another programme; this platform is
AR Enterprise's own. Formulas and logic may be reused, names may not.

**Suite name:** **ARES** (AR Energy Services). Code prefix: `ares-`.

**Pattern:** plain-English display name + short lowercase code. The code is used in URLs, folders,
file names and database values. Codes are one word, lowercase, no numbers, never changed once data
exists under them.

| New display name | Code | Replaces (old work, for reference only) | What it does |
|---|---|---|---|
| Plant Survey | `survey` | FOX (fox-kisem) | Machine-wise data capture, name-plate OCR, plant electrical hierarchy |
| Textile Machines | `textile` | Jet EFF | Jet dyeing / textile machine efficiency |
| Compressed Air | `air` | A-CMP | Compressor efficiency, leak tests, smart-nozzle cases |
| Pumps | `pump` | Pump-Up / pump efficiency | Pump tests (transit / Doppler), shaft power, efficiency, friction loss |
| Cooling | `cool` | Zero Degree | AC, chiller, AHU and cooling-tower performance |
| Boilers & Heaters | `heat` | Thermo-X | Boiler and thermic fluid heater efficiency (direct / indirect) |
| Power Quality | `power` | AI-PQA | PQ analyser file upload, harmonics, PF, anomaly checks |
| Bills | `bills` | PostMan bill reader | Electricity bill capture and verification |
| Solar | `solar` | — (new) | Solar farm audit for generation improvement |
| Report Studio | `report` | PostMan | Builds the energy audit report from stored data |
| Console | `admin` | — (new) | Users, roles, clients, audits, approvals, logs |
| Impact | `impact` | audit Website (KISEM dashboard) | Public totals, clusters, map |

Open question: "pump efficiency" and "PumpUp" were listed separately. Only one pump app was found
(`Pump-Up`). If there is a second one, it becomes a mode inside `pump`, not a new code.

Other naming rules:

- Repositories: `ares-<part>` (e.g. `ares-public`, `ares-ops`, `ares-api`) or one repo with folders
  of the same names — decide once, before the first code.
- Database tables and JSON keys: `snake_case`. TypeScript: `camelCase` variables, `PascalCase` types.
- Dates in files and IDs: `YYYY-MM-DD`. Financial year written as `FY2026-27`.
- Units always in the field name when ambiguous: `power_kw`, `flow_m3h`, `temp_c`.

---

## 4. Identifiers (all system-generated)

Nobody types an ID. The server creates them and they never change.

| Thing | Format | Example | Public? |
|---|---|---|---|
| Client (organisation) | `CL-` + 5 random base32 chars | `CL-7K2QD` | **Never** |
| Site (one plant) | `ST-` + 5 random chars | `ST-M4XP9` | **Never** |
| Audit (one engagement) | `AU-<FY>-<nnnn>` | `AU-2627-0012` | Never |
| Asset (one machine) | `<audit>-<code>-<nn>` | `AU-2627-0012-air-03` | Never |
| Record / reading | UUID | | Never |
| **Industry tag** (public) | `<SECTOR>-<CLUSTER>-<nnn>` | `TX-SRT-014` | Yes |

**Industry tag rules** — this is the only label the public ever sees for a plant:

- Generated by the server when an audit is first published. Sector and cluster come from fixed
  lists (below); the number is the next free one in that sector + cluster. It is **not** built from
  the company name, initials, GST number, address or anything else that could identify the client.
- Sector codes (extend only by adding to this table): `TX` textile, `CH` chemicals, `PH` pharma,
  `FD` food, `CR` ceramics, `EN` engineering, `PL` plastics, `PP` paper, `DY` dyes & pigments,
  `OT` other.
- Cluster codes: three letters of the city/industrial area, e.g. `SRT` Surat, `ANK` Ankleshwar,
  `VAP` Vapi, `MRB` Morbi, `AHD` Ahmedabad. Add new ones here.
- The link between a tag and the real client lives only in the private database.

---

## 5. Public data rules (what the public website may show)

The public Impact page is modelled on the earlier audit dashboard (map, totals, year-on-year
savings, sector charts) with these differences:

- **No client names, logos, addresses, contact people, photos of their plant, or exact
  coordinates** anywhere on the public site or in `impact.json`.
- Map points are placed at the **cluster centre**, not the plant. A cluster is shown only when it
  has **at least 3 published plants**; smaller groups are merged into "Other — <district>" so a
  single plant cannot be recognised.
- Each plant appears only as its industry tag, sector, cluster, audit year and its numbers
  (energy saved, cost saved, CO₂ avoided, measures identified / implemented).
- Only audits with status **Published** (see §7) count. Draft or unapproved data never reaches the
  public file.
- Every number shows its "as of" date. No estimated, rounded-up or invented figures, no fake
  reviews or testimonials.
- `impact.json` is produced by the server and copied to the public site; the public site never
  holds anything that is not already in that file.

---

## 6. How data flows between the tools

1. **Collect** — engineers use the field apps (`survey`, `textile`, `air`, `pump`, `cool`, `heat`,
   `power`, `bills`, `solar`). They work offline on the phone and sync when online.
2. **Store** — on sync, every record goes to the data server. Uploaded files (photos, analyser
   exports, bills, Excel) are saved under one folder layout:

   ```
   /data/<client-id>/<site-id>/<audit-id>/<module-code>/<YYYY-MM-DD>_<record-uuid>_v<n>.<ext>
   ```

   Files are never overwritten: an edit saves `v2`, `v3` … and keeps the old ones. A
   `manifest.json` per audit lists every file, who uploaded it, when, and its checksum.
3. **Process** — calculations run on the server from the stored readings, using the same formulas
   the old workbooks and apps used. Results are saved next to the inputs with the formula version.
4. **Report** — Report Studio builds the report only from what is stored for that audit ID. It does
   not accept emailed spreadsheets as a data source; anything from outside is uploaded first so it
   is stored and versioned like everything else.
5. **Publish** — when a report is approved, the server updates the anonymised totals in
   `impact.json`.

Data shape (shared by every tool):

```
Client → Site → Audit → Asset → Reading → Result → Measure (ECM) → Report
```

- Every module exports and imports one **canonical JSON** format, with `schema_version` and
  `module` fields at the top. Excel export stays available for people, but JSON is what the system
  trusts.
- The schemas live in one shared place (`schema/` folder or package) and every app uses them.
  Changing a field means bumping `schema_version` and updating Report Studio in the same change.
- Merging rules (carried over from the field apps): match by record UUID, newest save wins, older
  versions kept; an asset tag already used in the same audit is refused with the name of who used it.
- Every record carries: who recorded it, when it was taken, when it was last saved, and from which
  device.

---

## 7. Audit status (one lifecycle for everything)

`Draft → In field → Processing → Review → Approved → Published` (and `Archived`).

- Only a Head Member or Super Admin can move an audit to **Approved**.
- Only **Published** audits feed the public Impact page.
- Moving back from Approved/Published needs a written reason, which goes in the activity log.

---

## 8. Security for the Ops workspace and data server

Roles:

| Role | Can do |
|---|---|
| **Super Admin** | Everything, including managing Head Members, roles, system settings, backups, and deleting or archiving clients. Keep this to 1–2 people. |
| **Head Member** | All work in every module: create clients/sites/audits, collect and edit data, run processing, generate and approve reports, publish, manage Engineers. Cannot change Super Admins or system settings. |
| **Engineer** | Collect and edit data only for audits they are assigned to; generate draft reports. Cannot approve, publish, or see unassigned clients. |
| **Viewer** (optional, later) | Read-only access to assigned audits, e.g. a client's own report. |

Rules:

- No account is created by a seed script in production, and **no default passwords** (the old
  apps shipped seeded logins — none of those carry over). Super Admin creates every account and the
  person sets their own password on first login.
- Passwords stored only as strong hashes (argon2id or bcrypt). Two-factor login required for Super
  Admin and Head Member.
- Sessions in secure, HTTP-only cookies; automatic logout after inactivity; login attempts
  rate-limited and locked after repeated failures.
- Every action is checked on the **server** against the role and audit assignment — hiding a
  button is not security.
- Every create, edit, delete, export, download, approval and login is written to an **activity
  log** (who, what, when, from where). The log cannot be edited from the app.
- Files are only downloadable through short-lived signed links after a permission check. No public
  file URLs, no client names in any URL.
- Secrets (database passwords, mail keys, signing keys) live in server environment settings, never
  in GitHub. `.env` files are git-ignored.
- `ops.` and `api.` send `noindex` and are kept out of search engines.
- Daily database + file backups, kept off the main server, with a restore tested at least once a
  quarter.
- Field apps keep offline data encrypted on the phone and sign in again when the session expires.

---

## 9. What stays as it is right now

- This repository currently holds the **public website** (`index.html`, `assets/`). It is live and
  is not to be restructured until the build of the platform actually starts.
- When building starts, the public site moves into its part (`ares-public` / `public/`) without
  changing its look, and gains the Impact page described in §5.

## 10. Before any code is written — decisions still open

- [ ] Trial end date on Hostinger, and where the public site will live after it.
- [ ] Server choice for `ops` + `api` (Hostinger VPS or other) and database.
- [ ] One repository with folders, or separate `ares-*` repositories.
- [ ] Whether "pump efficiency" and "PumpUp" are one tool or two.
- [ ] Who the Super Admins are.
- [ ] First list of clusters and sectors to seed the industry-tag tables.
