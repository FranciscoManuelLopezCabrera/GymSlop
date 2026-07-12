# GymSlop
# Rutina Full Body — 3 días/semana (máquinas)

## Día A

| Ejercicio | Series x Reps | Backup 1 | Backup 2 |
|---|---|---|---|
| Prensa de piernas | 3x10-12 | Sentadilla Smith (guiada) | Sentadilla hack |
| Press de pecho en máquina | 3x8-10 | Press banca Smith (guiada) | Press pecho en polea |
| Jalón al pecho (polea) | 3x8-10 | Remo en máquina | Pull-over en polea |
| Press de hombro en máquina | 2x10-12 | Press hombro en polea | Press militar Smith |
| Curl femoral tumbado (máquina) | 3x10-12 | Curl femoral sentado | Peso muerto en máquina (glute drive) |
| Curl bíceps con mancuerna | 2x12-15 | Curl bíceps en polea | Curl bíceps en máquina |
| Cinta | 15 min | — | — |

## Día B

| Ejercicio | Series x Reps | Backup 1 | Backup 2 |
|---|---|---|---|
| Sentadilla hack (máquina) | 3x10-12 | Prensa de piernas | Sentadilla Smith |
| Press inclinado en máquina | 3x8-10 | Press inclinado Smith | Press pecho superior en polea |
| Remo en máquina (horizontal) | 3x8-10 | Remo en polea sentado | Jalón al pecho |
| Elevación lateral en polea/máquina | 3x12-15 | Elevación lateral en polea (agarre opuesto) | Máquina deltoides lateral |
| Hip thrust / glute drive (máquina) | 3x10-12 | Puente de glúteo en máquina | Prensa de piernas (pies altos) |
| Extensión tríceps en polea | 2x12-15 | Press francés en polea | Fondos en máquina asistida |
| Cinta | 15 min | — | — |

## Día C

| Ejercicio | Series x Reps | Backup 1 | Backup 2 |
|---|---|---|---|
| Prensa de piernas (pies altos, glúteo) | 3x10-12 | Hip thrust / glute drive (máquina) | Sentadilla Smith pies altos |
| Press de pecho agarre cerrado (máquina) | 3x8-10 | Press pecho en polea agarre cerrado | Fondos en máquina asistida |
| Remo en polea baja | 3x8-10 | Remo en máquina horizontal | Jalón al pecho agarre estrecho |
| Face pull en polea | 2x12-15 | Pájaros en polea | Remo a la cara en máquina |
| Curl femoral (máquina) | 3x10-12 | Peso muerto en máquina | Hip thrust / glute drive |
| Plancha / core | 3x30-45seg | Crunch en máquina | Rueda abdominal (si hay) |
| Cinta | 15 min | — | — |

## Notas

- La polea (cable) se cuenta como máquina — es guiada y fácil de controlar, no requiere técnica de peso libre.
- **Progresión:** cuando completes las 3 series en el rango alto de reps con buena forma, sube peso.
- **Distribución semanal:** lunes-miércoles-viernes (o día sí día no), mínimo 48h entre sesiones.

## Respaldo del diseño (revisado con evidencia)

- Frecuencia 3x/semana por grupo muscular: por encima del mínimo de 2x/semana que muestran los metaanálisis de Schoenfeld et al. para maximizar hipertrofia.
- Volumen semanal ~9-11 series por grupo muscular: dentro del rango de 10-20 series/semana que la mayoría de metaanálisis asocian a mejores ganancias (isquios/glúteo ajustado de 4 a 9 series tras la revisión).
- Rangos de 8-15 reps: sin diferencia real en hipertrofia frente a rangos más altos o más bajos si las series se acercan al fallo, así que no hay necesidad de cambiarlos.
- Máquinas/polea en vez de peso libre: un metaanálisis de 2023 (1016 sujetos) no encontró diferencias en ganancia de hipertrofia entre máquinas y peso libre — sí puede haber diferencias de fuerza específica del gesto, pero no en músculo ganado.

## App web (móvil)

App Vue 3 en un solo `index.html` (sin build). Pensada para el móvil en el gym.

- **Perfiles:** al entrar por primera vez, pantalla **¿Quién entrena?** para elegir/crear perfil. La elección se guarda en el navegador y se mantiene entre sesiones (no vuelve a preguntar). Cada persona tiene su rutina, pesos y calendario. Gestión en **⚙️ Ajustes** (crear / cambiar / borrar, o *🔀 Cambiar de perfil*). Perfil por defecto: **Paco** (Full Body).
- **4 tipos de rutina al crear perfil:**
  - **Full Body** — 3 días A/B/C con backups (la rutina base del README).
  - **Pull Push Legs** — 3 días tirón / empuje / pierna, con backups.
  - **Clases** — sin ejercicios; solo registra la asistencia a clase en el calendario.
  - **Personalizado** — empieza vacío; añade tus días y ejercicios a mano.
- **Editar rutina:** en Ajustes editas cada día: renombrar día, añadir/eliminar días y ejercicios; por ejercicio series, reps y peso. *Restaurar día* vuelve a la plantilla original (rutinas estándar).
- **Picklist de ejercicios:** el nombre se elige de un catálogo (67 ejercicios de máquina/polea) con **buscador rápido por nombre o grupo muscular** (acento-insensible: "biceps" encuentra "Bíceps"). Modo **✏️ Texto** para escribir un nombre propio.
- **Entreno:** pestañas Día A/B/C, marca cada serie, apunta el peso, barra de progreso.
- **Cambiar ejercicio:** los ejercicios de la rutina original traen 2 backups; si la máquina está ocupada, tocas y usas la alternativa.
- **Calendario:** registra qué día entrenaste (A/B/C) o descanso, con racha y contador mensual.
- **Timer de descanso:** arranca solo al marcar una serie (90 s por defecto, ±15 s).
- **PWA offline:** service worker + manifest → "Añadir a pantalla de inicio" y funciona sin cobertura.
- Todo se guarda en `localStorage` del móvil (clave `gymslop.v3`; migra automáticamente desde la versión anterior).

### Archivos

| Archivo | Qué es |
|---|---|
| `index.html` | App completa (Vue) |
| `vue.global.prod.js` | Vue 3 vendorizado (sin CDN, offline) |
| `manifest.json` / `icon.svg` | PWA |
| `sw.js` | Cache offline |

### Despliegue gratis — GitHub Pages

1. Sube el repo a GitHub.
2. **Settings → Pages → Source:** `Deploy from a branch`, rama `main`, carpeta `/ (root)`.
3. Guarda. En ~1 min queda en `https://<usuario>.github.io/GymSlop/`.
4. Ábrelo en el móvil → menú del navegador → *Añadir a pantalla de inicio*.

Alternativa sin git: arrastra la carpeta a [app.netlify.com/drop](https://app.netlify.com/drop).