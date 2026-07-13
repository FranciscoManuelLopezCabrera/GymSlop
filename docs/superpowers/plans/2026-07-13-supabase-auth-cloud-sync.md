# Supabase Auth + Cloud Sync Implementation Plan claude --resume 2bb421ff-2d59-4bb8-be21-53e578431233

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add email/password auth (register + login) and cloud persistence of routines to PacoGym via Supabase REST, so each user's data lives in the cloud and each user can have up to 4 profiles.

**Architecture:** Everything stays in the single `index.html`. No SDK — plain `fetch` against Supabase GoTrue (`/auth/v1`) and PostgREST (`/rest/v1`). One row per user in a `user_states` table holding `{active, profiles}` as `jsonb`. Login is required; the source of truth is the cloud. `localStorage` is used only to persist the auth session.

**Tech Stack:** Vue 3 (Options API, vendored, no build), Supabase (Postgres + GoTrue + PostgREST + RLS), browser `fetch`.

## Global Constraints

- Single `index.html`. **No build, no package.json, no external CDN, no Supabase SDK.** Only `fetch`. (from CLAUDE.md)
- Keep native/async objects out of Vue's reactive `data()` (module-level pattern, like `audioCtx`/`wakeLock`).
- Domain/UI text in **Spanish**. Dark theme, accent orange (`--acc:#ff6a2b`). Reuse existing CSS vars and component classes (`.chooser`, `.ch-box`, `.card-box`, `.btn`, `.btn.primary`, `.btn.danger`).
- **This repo has no unit-test tooling** (CLAUDE.md: "no build, lint, or test tooling"). Per-task verification is therefore: (a) JS **syntax check** via `node --check` on the extracted inline script, and (b) a concrete **manual/browser/curl** check. This mirrors the repo's existing workflow.
- After changing app assets, **bump `CACHE` in `sw.js`** (`gymslop-v13` → `gymslop-v14`).
- Max **4 profiles** per user, enforced in the app.

**Reusable syntax-check command** (used as the "run the test" step in code tasks — extracts only the bare inline `<script>` block, not the `vue.global.prod.js` tag):

```bash
awk '/^<script>$/{f=1;next} f&&/^<\/script>$/{f=0} f' index.html > /tmp/pg-app.js && node --check /tmp/pg-app.js && echo "JS SYNTAX OK"
```

---

## Task 0: Supabase project setup (manual, one-time)

**Files:** none (external service). Record completion in the spec's setup section.

This task has no code; it provisions the backend the later tasks talk to. It can be done in parallel with Tasks 1–6 (which only need the values pasted in Task 1), but the **end-to-end** verification in Task 7 depends on it.

- [ ] **Step 1: Create project.** At supabase.com create a free project. From Project Settings → API copy the **Project URL** and the **anon public** key. Keep them for Task 1.

- [ ] **Step 2: Disable email confirmation.** Authentication → Providers → Email: ensure Email is enabled and **"Confirm email" is OFF** (so `signup` returns a session immediately).

- [ ] **Step 3: Create table + RLS.** SQL Editor → run:

```sql
create table user_states (
  user_id    uuid primary key references auth.users(id) on delete cascade,
  data       jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);
alter table user_states enable row level security;
create policy "sel" on user_states for select using (auth.uid() = user_id);
create policy "ins" on user_states for insert with check (auth.uid() = user_id);
create policy "upd" on user_states for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

- [ ] **Step 4: Verify.** Table editor shows an empty `user_states`. Done.

---

## Task 1: Supabase config + `sb()` fetch helper

**Files:**
- Modify: `index.html` — insert after `releaseWake()` (the last module-level helper, currently ~line 1073) and before `createApp({`.

**Interfaces:**
- Produces: module consts `SB_URL`, `SB_KEY`, `AUTHKEY`; module async fn `sb(path, {method, body, token, prefer})` → parsed JSON (or `null`), throws `Error` with a human message on non-2xx.
- Consumed by: all later auth/cloud methods.

- [ ] **Step 1: Add config + helper.** Insert this block immediately after the `function releaseWake(){…}` line:

```js

// ---- Supabase (auth + cloud sync via REST, sin SDK) ----
const SB_URL = "https://TUPROYECTO.supabase.co";   // <-- pega tu Project URL (Task 0)
const SB_KEY = "PON_AQUI_TU_ANON_KEY";             // <-- pega tu anon public key (pública por diseño; RLS protege)
const AUTHKEY = "gymslop.auth";                    // localStorage: solo la sesión de auth
async function sb(path, { method="GET", body=null, token=null, prefer=null }={}){
  const h = { "apikey": SB_KEY, "Content-Type": "application/json" };
  if(token) h["Authorization"] = "Bearer " + token;
  if(prefer) h["Prefer"] = prefer;
  const res = await fetch(SB_URL + path, { method, headers:h, body: body?JSON.stringify(body):null });
  const txt = await res.text();
  const data = txt ? JSON.parse(txt) : null;
  if(!res.ok) throw new Error((data && (data.error_description||data.msg||data.message||data.error)) || ("HTTP "+res.status));
  return data;
}
```

- [ ] **Step 2: Paste real credentials.** Replace `SB_URL` and `SB_KEY` with the values from Task 0. (If Task 0 isn't done yet, leave placeholders — syntax still passes; e2e waits for Task 7.)

- [ ] **Step 3: Syntax check.**

Run: `awk '/^<script>$/{f=1;next} f&&/^<\/script>$/{f=0} f' index.html > /tmp/pg-app.js && node --check /tmp/pg-app.js && echo "JS SYNTAX OK"`
Expected: `JS SYNTAX OK`

- [ ] **Step 4: Commit.**

```bash
git add index.html
git commit -m "feat(auth): config Supabase + helper fetch sb()"
```

---

## Task 2: State scaffolding — auth fields, empty-profile fallback

Prepares `data()` so the app can start with **no profiles** (they arrive from the cloud) without the profile-dependent computeds throwing. `prof` gets a safe fallback; the app content stays hidden behind the auth/chooser overlays until data loads.

**Files:**
- Modify: `index.html` — `data()` (~1078–1103), the `prof()` computed (~1117), add module const near the other consts.

**Interfaces:**
- Produces: reactive fields `authed`, `loaded`, `user`, `authMode`, `authEmail`, `authPass`, `authErr`, `authBusy`, `syncErr`, `saveT`; module const `EMPTY_PROFILE`; `prof()` never returns undefined.
- Consumed by: Tasks 3–6.

- [ ] **Step 1: Add `EMPTY_PROFILE` const.** Right after the `sb()` block from Task 1, add:

```js
const EMPTY_PROFILE = { type:"full", days:[], sets:{}, swaps:{}, log:{}, week:{}, rest:90 };
```

- [ ] **Step 2: Start with empty state + auth fields in `data()`.** In `data()`, delete the `const st = loadState();` line. Change the returned `active` and `profiles` to start empty, and add the auth/sync fields. Replace:

```js
      active: st.active,
      profiles: st.profiles,
```
with:
```js
      active: "",
      profiles: {},
```
Then, immediately after the `session: false, sesIdx: 0, sesPickOpen: false,` line, add:
```js
      authed: false, loaded: false, user: null,
      authMode: "login", authEmail: "", authPass: "", authErr: "", authBusy: false,
      syncErr: false, saveT: null,
```

- [ ] **Step 3: Make `prof()` fallback-safe.** Replace the `prof()` computed:

```js
    prof(){ return this.profiles[this.active] || this.profiles[Object.keys(this.profiles)[0]]; },
```
with:
```js
    prof(){ return this.profiles[this.active] || this.profiles[Object.keys(this.profiles)[0]] || EMPTY_PROFILE; },
```

- [ ] **Step 4: Syntax check.**

Run: `awk '/^<script>$/{f=1;next} f&&/^<\/script>$/{f=0} f' index.html > /tmp/pg-app.js && node --check /tmp/pg-app.js && echo "JS SYNTAX OK"`
Expected: `JS SYNTAX OK`

- [ ] **Step 5: Commit.**

```bash
git add index.html
git commit -m "feat(auth): estado inicial vacío + fallback de perfil"
```

---

## Task 3: Cloud load/save + watchers + max-4 profiles

Adds the cloud read/write and repoints persistence from localStorage to Supabase. Debounced saves; guarded so the empty startup state never overwrites the cloud.

**Files:**
- Modify: `index.html` — add methods after `finishSession()` (in the training-session method block); replace the `watch` block (~1372–1373); remove the `persist()` method (~1179); add a guard in `addProfile()` (~1260).

**Interfaces:**
- Consumes: `sb()`, `EMPTY_PROFILE`, `makeProfile()`, `this.user`, `this.refreshIfNeeded()` (defined in Task 4 — forward reference; both land before any runtime call).
- Produces: `loadCloud()`, `saveCloud()`, `queueSave()`.

- [ ] **Step 1: Add cloud methods.** After the `finishSession(){ … },` line, add:

```js
    // cloud sync (Supabase)
    async loadCloud(){
      const token = await this.refreshIfNeeded();
      const rows = await sb("/rest/v1/user_states?user_id=eq."+this.user.id+"&select=data", { token });
      if(rows && rows.length && rows[0].data && rows[0].data.profiles){
        this.active = rows[0].data.active || "";
        this.profiles = rows[0].data.profiles;
      } else {
        this.profiles = { "Paco": makeProfile("full") };
        this.active = "";
        this.loaded = true;          // permite el primer saveCloud (insert)
        await this.saveCloud();
      }
      this.syncDays();
      this.loaded = true;
    },
    async saveCloud(){
      if(!this.user) return;
      try{
        const token = await this.refreshIfNeeded();
        await sb("/rest/v1/user_states", {
          method:"POST",
          prefer:"resolution=merge-duplicates,return=minimal",
          token,
          body:{ user_id:this.user.id, data:{ active:this.active, profiles:this.profiles }, updated_at:new Date().toISOString() }
        });
        this.syncErr = false;
      }catch(e){ this.syncErr = true; }
    },
    queueSave(){
      if(!this.loaded) return;
      clearTimeout(this.saveT);
      this.saveT = setTimeout(()=>this.saveCloud(), 800);
    },
```

- [ ] **Step 2: Repoint watchers to `queueSave`.** Replace:

```js
    active: "persist",
    profiles: { deep: true, handler: "persist" },
```
with:
```js
    active: "queueSave",
    profiles: { deep: true, handler: "queueSave" },
```

- [ ] **Step 3: Remove the now-unused `persist()` method.** Delete the line:

```js
    persist(){ localStorage.setItem(KEY, JSON.stringify({ active:this.active, profiles:this.profiles })); },
```

- [ ] **Step 4: Enforce max 4 profiles.** In `addProfile()`, insert as the first statement inside the method (before `const n=this.newProfile.trim();`):

```js
      if(Object.keys(this.profiles).length>=4){ alert("Máximo 4 perfiles."); return; }
```

- [ ] **Step 5: Syntax check.**

Run: `awk '/^<script>$/{f=1;next} f&&/^<\/script>$/{f=0} f' index.html > /tmp/pg-app.js && node --check /tmp/pg-app.js && echo "JS SYNTAX OK"`
Expected: `JS SYNTAX OK`

- [ ] **Step 6: Commit.**

```bash
git add index.html
git commit -m "feat(sync): carga/guardado en la nube + watchers + max 4 perfiles"
```

---

## Task 4: Auth methods + session restore on mount

**Files:**
- Modify: `index.html` — add methods after `chooseEx()` / near the profile methods (place right before `startCreate()`); extend `mounted()` (~1104–1110).

**Interfaces:**
- Consumes: `sb()`, `AUTHKEY`, `this.loadCloud()`, `this.doLogout()`.
- Produces: `saveSession(resp)`, `readSession()`, `refreshIfNeeded()`→`Promise<access_token>`, `register()`, `login()`, `doLogout()`, `restoreSession()`.

- [ ] **Step 1: Add auth methods.** Insert immediately before `startCreate(){…}`:

```js
    // auth (Supabase GoTrue)
    saveSession(resp){
      const now = Math.floor(Date.now()/1000);
      const sess = {
        access_token: resp.access_token,
        refresh_token: resp.refresh_token,
        expires_at: now + (resp.expires_in || 3600),
        user: resp.user ? { id:resp.user.id, email:resp.user.email } : (this.user || {})
      };
      localStorage.setItem(AUTHKEY, JSON.stringify(sess));
      this.user = sess.user;
      return sess;
    },
    readSession(){ try{ return JSON.parse(localStorage.getItem(AUTHKEY)); }catch(e){ return null; } },
    async refreshIfNeeded(){
      const s = this.readSession(); if(!s) throw new Error("Sesión no encontrada.");
      const now = Math.floor(Date.now()/1000);
      if(s.expires_at - now > 60) return s.access_token;
      const r = await sb("/auth/v1/token?grant_type=refresh_token", { method:"POST", body:{ refresh_token:s.refresh_token } });
      return this.saveSession(r).access_token;
    },
    async register(){
      const email=this.authEmail.trim(), pass=this.authPass;
      if(!email || pass.length<6){ this.authErr="Email válido y contraseña de 6+ caracteres."; return; }
      this.authErr=""; this.authBusy=true;
      try{
        const r = await sb("/auth/v1/signup", { method:"POST", body:{ email, password:pass } });
        if(!r.access_token) throw new Error("Revisa tu email para confirmar la cuenta.");
        this.saveSession(r); this.authed=true; this.authPass="";
        await this.loadCloud();
      }catch(e){ this.authErr = e.message || "No se pudo crear la cuenta."; }
      finally{ this.authBusy=false; }
    },
    async login(){
      const email=this.authEmail.trim(), pass=this.authPass;
      if(!email || !pass){ this.authErr="Introduce email y contraseña."; return; }
      this.authErr=""; this.authBusy=true;
      try{
        const r = await sb("/auth/v1/token?grant_type=password", { method:"POST", body:{ email, password:pass } });
        this.saveSession(r); this.authed=true; this.authPass="";
        await this.loadCloud();
      }catch(e){ this.authErr = e.message || "Email o contraseña incorrectos."; }
      finally{ this.authBusy=false; }
    },
    doLogout(){
      const s=this.readSession();
      if(s){ sb("/auth/v1/logout", { method:"POST", token:s.access_token }).catch(()=>{}); }
      localStorage.removeItem(AUTHKEY);
      this.authed=false; this.loaded=false; this.user=null;
      this.profiles={}; this.active=""; this.authEmail=""; this.authPass=""; this.authErr=""; this.syncErr=false;
    },
    async restoreSession(){
      const s = this.readSession(); if(!s) return;
      this.user = s.user;
      try{ await this.refreshIfNeeded(); this.authed=true; await this.loadCloud(); }
      catch(e){ this.doLogout(); }
    },
```

- [ ] **Step 2: Restore session on mount.** In `mounted()`, add as the first statement (before `this.syncDays();`):

```js
    this.restoreSession();
```

- [ ] **Step 3: Syntax check.**

Run: `awk '/^<script>$/{f=1;next} f&&/^<\/script>$/{f=0} f' index.html > /tmp/pg-app.js && node --check /tmp/pg-app.js && echo "JS SYNTAX OK"`
Expected: `JS SYNTAX OK`

- [ ] **Step 4: Commit.**

```bash
git add index.html
git commit -m "feat(auth): register/login/logout + restaurar sesión al arrancar"
```

---

## Task 5: Auth gate UI + chooser gating + CSS

Shows a login/register screen when logged out and a loading screen while cloud data loads; keeps the existing profile chooser hidden until authenticated and loaded.

**Files:**
- Modify: `index.html` — insert auth overlays right after `<div id="app" v-cloak>` (line 307); gate the chooser `v-if` (line 309); append CSS before `</style>`.

**Interfaces:**
- Consumes: `authed`, `loaded`, `authMode`, `authEmail`, `authPass`, `authErr`, `authBusy`, `login()`, `register()`.

- [ ] **Step 1: Insert auth + loading overlays.** Immediately after `<div id="app" v-cloak>` add:

```html
  <!-- AUTH GATE -->
  <div v-if="!authed" class="chooser auth">
    <div class="ch-box">
      <div class="ch-logo">🏋️</div>
      <h2>Paco<span>Gym</span><small class="beta">Beta</small></h2>
      <p class="ch-sub">{{ authMode==='login' ? 'Entra en tu cuenta' : 'Crea tu cuenta' }}</p>
      <div class="auth-tabs">
        <button :class="{on:authMode==='login'}" @click="authMode='login'; authErr=''">Entrar</button>
        <button :class="{on:authMode==='register'}" @click="authMode='register'; authErr=''">Crear cuenta</button>
      </div>
      <input class="auth-in" type="email" inputmode="email" autocomplete="email" v-model="authEmail"
             placeholder="Email" @keyup.enter="authMode==='login' ? login() : register()">
      <input class="auth-in" type="password" autocomplete="current-password" v-model="authPass"
             placeholder="Contraseña" @keyup.enter="authMode==='login' ? login() : register()">
      <p v-if="authErr" class="auth-err">{{ authErr }}</p>
      <button class="btn primary auth-go" :disabled="authBusy" @click="authMode==='login' ? login() : register()">
        {{ authBusy ? 'Conectando…' : (authMode==='login' ? 'Entrar' : 'Crear cuenta') }}
      </button>
    </div>
  </div>
  <div v-else-if="!loaded" class="chooser auth">
    <div class="ch-box"><div class="ch-logo">🏋️</div><p class="ch-sub">Cargando tus rutinas…</p></div>
  </div>

```

- [ ] **Step 2: Gate the profile chooser behind auth.** Replace:

```html
  <div v-if="needsPick || picking" class="chooser">
```
with:
```html
  <div v-if="authed && loaded && (needsPick || picking)" class="chooser">
```

- [ ] **Step 3: Append CSS.** Before `</style>` add:

```css
  /* auth gate */
  .auth-tabs{display:flex; gap:8px; margin:2px 0 14px}
  .auth-tabs button{flex:1; padding:10px; border-radius:10px; border:1px solid var(--line); background:var(--card2); color:var(--mut); font-weight:700; font-size:13px; cursor:pointer}
  .auth-tabs button.on{background:linear-gradient(135deg,var(--acc2),var(--acc)); color:var(--ink); border-color:transparent}
  .auth-in{width:100%; padding:13px; margin-bottom:10px; border-radius:11px; border:1px solid var(--line); background:var(--card2); color:var(--txt); font-size:15px; box-sizing:border-box}
  .auth-err{color:var(--red); font-size:13px; margin:2px 0 10px; text-align:center}
  .auth-go{width:100%}
```

- [ ] **Step 4: Verify in browser (logged-out render).** Serve and open the app; confirm the login screen shows and does not crash.

```bash
python -m http.server 8000
```
Open `http://localhost:8000/` in a **private/incognito** window (no session). Expected: the **Entrar / Crear cuenta** screen renders (orange tabs, email + contraseña inputs, button). Toggling tabs switches the button/subtitle. DevTools console shows no errors. (No real submit yet unless Task 0 creds are in.)

- [ ] **Step 5: Commit.**

```bash
git add index.html
git commit -m "feat(auth): pantalla login/registro + gate del selector de perfil"
```

---

## Task 6: Ajustes account card + service-worker cache bump

**Files:**
- Modify: `index.html` — settings `<main>` (line 489); append CSS before `</style>`.
- Modify: `sw.js` — line 2 cache name.

**Interfaces:**
- Consumes: `user`, `syncErr`, `doLogout()`.

- [ ] **Step 1: Add the "Cuenta" section at the top of Ajustes.** Replace:

```html
  <main v-if="view==='settings'">
    <h3 class="sec">Perfil</h3>
```
with:
```html
  <main v-if="view==='settings'">
    <h3 class="sec">Cuenta</h3>
    <div class="card-box acct">
      <div class="acct-row">
        <span>{{ user ? user.email : '' }}</span>
        <em :class="{bad:syncErr}">{{ syncErr ? '⚠ sin conexión' : 'guardado ✓' }}</em>
      </div>
      <button class="btn danger" style="width:100%" @click="doLogout">Cerrar sesión</button>
    </div>
    <h3 class="sec">Perfil</h3>
```

- [ ] **Step 2: Append CSS.** Before `</style>` add:

```css
  /* cuenta */
  .acct-row{display:flex; align-items:center; justify-content:space-between; gap:10px; margin-bottom:10px; font-size:14px; color:var(--txt)}
  .acct-row em{font-style:normal; font-size:12px; color:var(--green); font-weight:700; white-space:nowrap}
  .acct-row em.bad{color:var(--red)}
```

- [ ] **Step 3: Bump the SW cache.** In `sw.js` replace:

```js
const CACHE = "gymslop-v13";
```
with:
```js
const CACHE = "gymslop-v14";
```

- [ ] **Step 4: Syntax check + serve.**

Run: `awk '/^<script>$/{f=1;next} f&&/^<\/script>$/{f=0} f' index.html > /tmp/pg-app.js && node --check /tmp/pg-app.js && echo "JS SYNTAX OK"`
Expected: `JS SYNTAX OK`

- [ ] **Step 5: Commit.**

```bash
git add index.html sw.js
git commit -m "feat(auth): tarjeta de cuenta en Ajustes + bump SW v14"
```

---

## Task 7: End-to-end acceptance (requires Task 0 done + creds in Task 1)

**Files:** none (verification only). If any check fails, fix the relevant task's code and re-run.

Serve locally and drive the full flow against the real Supabase project:

```bash
python -m http.server 8000
```
Open `http://localhost:8000/` (never `file://`).

- [ ] **Step 1: Register.** On the auth screen choose **Crear cuenta**, enter a test email + password (6+ chars), submit. Expected: enters the app directly (confirm OFF); the profile chooser shows **Paco**. In Supabase → Table editor → `user_states` a row now exists with your `user_id` and `data.profiles.Paco`.

- [ ] **Step 2: Save syncs to cloud.** Pick Paco, edit a weight / mark a set / edit the routine. Wait ~1s. Ajustes shows **guardado ✓**. Refresh the `user_states` row → `data` reflects the change and `updated_at` advanced.

- [ ] **Step 3: Max 4 profiles.** Create profiles until there are 4, then try a 5th. Expected: `alert("Máximo 4 perfiles.")` and no 5th profile is created.

- [ ] **Step 4: Logout / login round-trip.** Ajustes → **Cerrar sesión** → auth screen. **Entrar** with the same credentials → the exact same profiles/data load from the cloud.

- [ ] **Step 5: Cross-context.** Open the app in a different browser (or another incognito window) and log in with the same account → identical data appears (proves it lives in the cloud, not the device).

- [ ] **Step 6: Offline indicator.** In DevTools → Network set **Offline**, edit something → after ~1s Ajustes shows **⚠ sin conexión**. Set back **Online**, edit again → returns to **guardado ✓**.

- [ ] **Step 7: Bad credentials.** Logout, try logging in with a wrong password → the auth screen shows an error message and does not enter.

- [ ] **Step 8: Session persistence.** While logged in, refresh the page → it restores the session and loads data without asking to log in again.

---

## Self-Review Notes

- **Spec coverage:** login/register (Tasks 4–5), Supabase config (Task 1), `user_states` blob + RLS (Tasks 0, 3), solo-nube + login gate (Tasks 2, 5), email confirm OFF (Task 0), start-clean seed Paco (Task 3 `loadCloud`), max 4 (Task 3), account/logout UI + sync indicator (Task 6), SW bump (Task 6), verification (Task 7). All covered.
- **Forward reference:** Task 3's `loadCloud`/`saveCloud` call `this.refreshIfNeeded()` defined in Task 4. Both are methods on the same instance and no code path runs before both exist (first call is via `restoreSession`/`login`, wired in Task 4). Implement Tasks 3 and 4 before any Task 7 run.
- **Naming consistency:** `SB_URL`, `SB_KEY`, `AUTHKEY`, `sb()`, `EMPTY_PROFILE`, `saveSession`, `readSession`, `refreshIfNeeded`, `loadCloud`, `saveCloud`, `queueSave`, `saveT`, `authed`, `loaded`, `syncErr` used consistently across tasks.
- **No placeholders** except `SB_URL`/`SB_KEY`, which are intentional per-deployment secrets filled in Task 1 Step 2 / Task 0.
