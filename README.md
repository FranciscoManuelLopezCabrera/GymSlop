# PacoGym (Beta)

App web (PWA) para consultar y registrar tu rutina de gym desde el móvil. Vue 3 en un solo `index.html`, **sin build ni dependencias externas**, desplegable en cualquier hosting estático gratuito.

## Funciones

- **Perfiles multiusuario:** varias personas, cada una con su rutina, pesos y calendario propios. Al entrar la primera vez eliges/creas perfil; la elección se guarda en el navegador y se mantiene entre sesiones (no vuelve a preguntar). Gestión en **⚙️ Ajustes** (crear / cambiar / borrar).
- **4 tipos de rutina** al crear perfil (ver tabla abajo).
- **Editor de rutina:** por día, renombrar día, añadir/eliminar días y ejercicios; por ejercicio series, reps y peso.
- **Picklist de ejercicios:** el nombre se elige de un catálogo de 160+ ejercicios (máquina, polea y **peso libre**: barra, mancuernas, kettlebell, peso corporal), con **buscador por nombre o grupo muscular** (acento-insensible: "biceps" encuentra "Bíceps"). Modo **✏️ Texto** para nombres personalizados.
- **Cambiar ejercicio en caliente:** las rutinas estándar traen 2 backups por ejercicio; si la máquina está ocupada, tocas y usas la alternativa.
- **Entreno:** marca cada serie, apunta el peso, barra de progreso, **timer de descanso** (arranca solo al marcar serie; 90 s, ±15 s, beep + vibración).
- **Calendario:** registra qué día entrenaste (o descanso), con racha, contador mensual y colores por día. **Plantilla semanal:** configuras una semana tipo y la replicas en todo el mes o el año de golpe; luego ajustas los días sueltos.
- **PWA:** funciona **offline** (service worker) y se **instala** en la pantalla de inicio (botón en Ajustes y en el selector de perfil).
- Todo se guarda en `localStorage` del dispositivo (clave `gymslop.v4`; migra solo desde versiones anteriores).

## Tipos de rutina

| Tipo | Días | Contenido |
|---|---|---|
| **Full Body** | A / B / C | Full body 3 días/semana con backups |
| **Pull Push Legs** | Pull / Push / Piernas | tirón / empuje / pierna, con backups |
| **Clases** | Clase | sin ejercicios; solo registra asistencia en el calendario |
| **Personalizado** | Día 1 | empieza vacío; creas tus días y ejercicios a mano |

## Instalar en el móvil

En **⚙️ Ajustes** (o en el selector de perfil) → botón **📲 Añadir a pantalla de inicio**.

- **Android (Chrome):** lanza el instalador nativo directamente.
- **iPhone/iPad (Safari):** muestra los pasos → Compartir ⤴ → *Añadir a pantalla de inicio*.

Una vez instalada, abre a pantalla completa y funciona sin cobertura.

## Archivos

| Archivo | Qué es |
|---|---|
| `index.html` | App completa (Vue 3, estilos y lógica inline) |
| `vue.global.prod.js` | Vue 3 vendorizado (sin CDN, offline) |
| `manifest.json` / `icon.svg` | Metadatos PWA + icono |
| `sw.js` | Service worker (cache offline) |
| `.nojekyll` | Sirve los archivos tal cual en GitHub Pages |

## Despliegue (gratis) — GitHub Pages

1. Sube el repo a GitHub.
2. **Settings → Pages → Source:** `Deploy from a branch`, rama `main`, carpeta `/ (root)`.
3. Guarda. En ~1 min queda en `https://<usuario>.github.io/GymSlop/`.

Alternativa sin git: arrastra la carpeta a [app.netlify.com/drop](https://app.netlify.com/drop).

> Al publicar una versión nueva, el service worker sube de número de cache (`gymslop-vN`). En el móvil, cierra y reabre la PWA para cargar el build actualizado.

## Tecnología

Vue 3 (build global vendorizado), CSS y JS inline, `localStorage` para persistencia, service worker + web manifest para PWA. Cero backend, cero cuentas, cero tracking.
