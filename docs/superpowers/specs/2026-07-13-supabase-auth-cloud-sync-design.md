# Spec — Auth + sincronización en la nube con Supabase

Fecha: 2026-07-13
App: PacoGym (Beta) — PWA Vue 3, un solo `index.html`, sin build, sin package.json, sin CDN externa.

## Contexto y objetivo

Hoy la app guarda todo en `localStorage["gymslop.v4"]` con forma `{active, profiles:{<nombre>:Profile}}`,
soporta varios perfiles por dispositivo y funciona 100% offline. Se quiere que los usuarios **se registren
y se logueen**, y que sus rutinas se guarden en una **base de datos gratuita (Supabase)** ligada a su cuenta.
Cada usuario puede tener **varios perfiles (máx. 4 de momento)**. Hay que añadir **login + registro +
configuración de Supabase**.

Resultado buscado: al abrir la app pide login/registro; tras entrar, los datos se cargan desde la nube y
cada cambio se guarda en la nube. Sin depender de la cuenta, los datos ya no viven solo en el dispositivo.

## Decisiones (cerradas con el usuario)

1. **Modelo de sincronización: solo nube, login obligatorio.** La app carga y guarda directo en Supabase.
   Sin sesión no se entra. Se acepta perder el uso offline actual de datos (el shell PWA sigue cacheado).
2. **Modelo de datos: blob JSON por usuario.** Una tabla `user_states` con una fila por usuario que contiene
   `{active, profiles}` en una columna `jsonb`. Refleja el `persist()` actual → un solo upsert al guardar.
3. **Confirmación de email: OFF.** Registro entra directo (entorno de pruebas). Se configura en Supabase.
4. **Migración: empezar limpio.** Se ignora el `localStorage` como fuente. Cuenta nueva → siembra un perfil
   por defecto `"Paco"` (Full Body). El máx. 4 perfiles se controla en la app.

## Restricciones técnicas (de CLAUDE.md, se mantienen)

- Un único `index.html`. **Sin build, sin package.json, sin CDN externa, sin SDK de Supabase.**
- Se usa **`fetch` puro** contra los endpoints REST de Supabase (GoTrue `/auth/v1`, PostgREST `/rest/v1`).
- Objetos nativos fuera de la reactividad de Vue (patrón ya usado con `audioCtx`/`wakeLock`/`installEvent`).
- Tras cambiar assets, **bump de `CACHE` en `sw.js`** (v13 → v14).

## Arquitectura

### Configuración (consts de nivel app, arriba del `<script>`)
```js
const SB_URL = "https://TUPROYECTO.supabase.co";   // rellenar tras crear el proyecto
const SB_KEY = "eyJ...anon-public...";             // anon key: pública por diseño, RLS protege los datos
```
Estos valores los rellena el desarrollador una vez. No hay UI para configurarlos (no son por-usuario).

### Helper de red
```js
async function sb(path, { method="GET", body=null, token=null, prefer=null }={}){
  const h = { "apikey": SB_KEY, "Content-Type": "application/json" };
  if(token) h["Authorization"] = "Bearer " + token;
  if(prefer) h["Prefer"] = prefer;
  const res = await fetch(SB_URL + path, { method, headers:h, body: body?JSON.stringify(body):null });
  const txt = await res.text();
  const data = txt ? JSON.parse(txt) : null;
  if(!res.ok) throw new Error((data && (data.error_description||data.msg||data.message)) || ("HTTP "+res.status));
  return data;
}
```

### Sesión
- Se guarda en `localStorage["gymslop.auth"]` = `{access_token, refresh_token, expires_at, user:{id,email}}`.
  (Único uso de localStorage; la fuente de datos de rutinas es la nube.)
- `expires_at` en epoch segundos; refresh proactivo si quedan <60s.

## Autenticación (GoTrue REST)

| Acción | Endpoint | Notas |
|---|---|---|
| Registro | `POST /auth/v1/signup` `{email,password}` | Con confirm OFF devuelve sesión al instante |
| Login | `POST /auth/v1/token?grant_type=password` `{email,password}` | |
| Refresh | `POST /auth/v1/token?grant_type=refresh_token` `{refresh_token}` | Cuando el token expira/va a expirar |
| Logout | limpiar sesión local + `POST /auth/v1/logout` (Bearer) | El logout de red es best-effort |

Respuesta de signup/login trae `access_token`, `refresh_token`, `expires_in`, `user{id,email}`.
`saveSession(resp)` calcula `expires_at = now + expires_in` y persiste el objeto.

## Datos (PostgREST)

### Esquema + RLS (SQL a ejecutar una vez en Supabase → SQL Editor)
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

### Cargar (`loadCloud`)
- `GET /rest/v1/user_states?user_id=eq.{uid}&select=data` (Bearer access_token).
- Si la fila existe → `this.active/this.profiles = row.data`.
- Si no existe (array vacío) → siembra `{active:"", profiles:{ "Paco": makeProfile("full") }}` y hace un
  insert inmediato (`saveCloud`). Luego `syncDays()`.
- Al terminar: `this.loaded = true`.

### Guardar (`saveCloud` / `queueSave`)
- Upsert: `POST /rest/v1/user_states` con `Prefer: "resolution=merge-duplicates,return=minimal"`,
  body `{ user_id, data:{active:this.active, profiles:this.profiles}, updated_at: <ISO> }`.
  (`updated_at` se sella con un timestamp pasado desde el runtime, no con `Date.now()` en scripts restringidos —
  aquí es la app normal en navegador, `new Date().toISOString()` es válido.)
- `queueSave()` = debounce ~800ms → `saveCloud`. Se invoca desde los watchers **solo si `this.loaded`**
  (evita pisar la nube con estado vacío durante el arranque).
- Fallo de red → `syncErr = true` (indicador "sin conexión"); reintenta en el siguiente cambio. Éxito → `syncErr=false`.

## Flujo en la app (Vue)

### `data()` — añadidos
```js
authed: false, loaded: false,
user: null,                       // {id,email}
authMode: "login",                // "login" | "register"
authEmail: "", authPass: "",
authErr: "", authBusy: false,
syncErr: false,
```
`profiles` arranca vacío `{}` y `active:""` (ya no se siembra desde `loadState()`); se llenan tras `loadCloud`.

### `mounted()`
1. `restoreSession()` lee `localStorage["gymslop.auth"]`. Si hay sesión:
   - refresca si hace falta; `this.user = sess.user; this.authed = true; await this.loadCloud()`.
   - si el refresh falla → `doLogout()` (a pantalla login).
2. Si no hay sesión → queda en pantalla auth (`authed=false`).
3. El resto del `mounted` actual (install prompt, visibilitychange) se conserva.

### Métodos nuevos (junto a los de perfiles)
`saveSession`, `restoreSession`, `refreshIfNeeded`, `register`, `login`, `doLogout`,
`loadCloud`, `saveCloud`, `queueSave`. `register`/`login` validan campos, ponen `authBusy`,
capturan error en `authErr`, y en éxito → `saveSession` + `loadCloud` + `authed=true`.

### Watchers
```js
active:  "queueSave",
profiles:{ deep:true, handler:"queueSave" },
```
Sustituyen a `persist`. `queueSave` no hace nada si `!this.loaded`.
`persist()`/`loadState()` de localStorage se retiran del camino de datos (se puede borrar `loadState` o dejarlo
sin usar; `makeProfile` sigue siendo la semilla por defecto).

### Máx 4 perfiles
`addProfile()` al principio: `if(Object.keys(this.profiles).length>=4){ alert("Máximo 4 perfiles."); return; }`.

## UI

### Pantalla de auth (gate)
Bloque de nivel superior mostrado cuando `!authed`, reutilizando estilos `.chooser`/`.card`/`.btn`/`.field`:
- Título/logo "PacoGym".
- Conmutador **Entrar / Crear cuenta** (`authMode`).
- Inputs email (`type=email`) y contraseña (`type=password`).
- Botón primario (`login`/`register`), deshabilitado si `authBusy`.
- `<p class="err" v-if="authErr">{{ authErr }}</p>`.
- Texto "Conectando…" mientras `authBusy`.

El resto de la app (`#app` contenido actual: picker de perfil, entreno, calendario, ajustes, modo
entrenamiento) se envuelve/gate con `v-if="authed && loaded"`. El picker de perfil existente
(`needsPick`/`picking`) sigue funcionando **por encima** para elegir entre los ≤4 perfiles.

### Ajustes
- Línea con el email logueado (`user.email`).
- Botón **Cerrar sesión** → `doLogout()`.
- Indicador de sync: "Guardado ✓" o "Sin conexión" según `syncErr`.

### CSS
Añadir estilos para `.auth`, `.auth-tabs`, `.field`, `.err` reutilizando variables `:root`
(`--acc`, `--card`, `--line`, `--mut`, `--rad`). Sin dependencias nuevas.

## Errores y casos límite

- **Credenciales inválidas / email ya registrado / password corta:** mensaje de GoTrue en `authErr`.
- **Sin red al entrar:** login/registro fallan con "sin conexión"; no se entra.
- **Token expirado:** `refreshIfNeeded` lo renueva de forma transparente antes de cada `loadCloud`/`saveCloud`;
  si el refresh falla → `doLogout()`.
- **Guardado sin red:** `syncErr=true`, se reintenta al siguiente cambio; no se pierde el estado en memoria.
- **Arranque (carrera):** `queueSave` bloqueado hasta `loaded=true` para no sobrescribir la nube con `{}`.
- **Cuenta nueva:** siembra `Paco` y hace insert inmediato (fila garantizada antes del primer upsert).
- **`anon key` en cliente:** pública por diseño; RLS (`auth.uid()=user_id`) impide leer/escribir filas ajenas.
  No se expone ningún secreto de servicio.

## Ficheros afectados

- `index.html` — consts `SB_URL`/`SB_KEY`, helper `sb()`, `saveSession`/`restoreSession`/`refreshIfNeeded`,
  `register`/`login`/`doLogout`, `loadCloud`/`saveCloud`/`queueSave`; cambios en `data()`, `mounted()`,
  `watch`, `addProfile`; pantalla auth + gate `v-if="authed && loaded"`; línea/botón de sesión en Ajustes; CSS.
- `sw.js` — bump `CACHE` `gymslop-v13` → `gymslop-v14`.
- Este spec documenta también el **setup de Supabase** (SQL + apagar confirm email).

## Setup de Supabase (pasos manuales del desarrollador)

1. Crear proyecto gratis en supabase.com; copiar **Project URL** y **anon public key** → pegarlos en
   `SB_URL`/`SB_KEY`.
2. Authentication → Providers → **Email**: activar Email, **Confirm email = OFF**.
3. SQL Editor: ejecutar el bloque `create table user_states … policies` de arriba.
4. (Opcional) Authentication → URL config: no crítico con confirm OFF.

## Verificación (extremo a extremo, sobre HTTP)

Servir local: `python -m http.server 8000` → `http://localhost:8000/` (nunca `file://`). Con `SB_URL`/`SB_KEY`
puestos y el SQL ejecutado:
1. Abrir sin sesión → aparece pantalla **auth**.
2. **Crear cuenta** (email+pass) → entra directo (confirm OFF); se crea fila `user_states` con perfil `Paco`.
3. Editar rutina / marcar series / cambiar peso → tras ~1s aparece "Guardado ✓"; en Supabase → Table editor
   se ve `data` actualizado.
4. Crear perfiles hasta 4; el **5º** se bloquea con "Máximo 4 perfiles".
5. **Cerrar sesión** → vuelve a auth. **Entrar** de nuevo → se recuparan exactamente los datos de la nube.
6. Abrir en **otro navegador/incógnito** con la misma cuenta → mismos datos (confirma que viven en la nube).
7. Cortar red y editar → indicador "Sin conexión"; reconectar y editar → vuelve a "Guardado ✓".
8. Credenciales incorrectas → mensaje de error en la pantalla auth.
