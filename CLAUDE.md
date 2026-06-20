# CLAUDE.md — Quiniela Web App (Liga de Fútbol)

> Archivo principal de contexto para **Claude Code**.
> **Misión del producto:** construir una **web app** cuyo objetivo es **administrar el juego de quiniela de una liga de fútbol**: los participantes pronostican los resultados de las jornadas, el sistema cierra pronósticos en la fecha límite, calcula puntos contra los resultados oficiales y mantiene una **tabla de posiciones** de participantes.
> **Arquitectura:** frontend estático en **GitHub Pages** + **Google Sheets como base de datos**. Autenticación vía **Google Auth** (principal, con número de celular verificado) y **correo electrónico** como alternativa.
> **Base visual (UI/UX):** se replica el estilo de la captura de referencia (app deportiva oscura tipo FlashScore) — encabezado con menú desplegable, navegación por fechas/jornadas, espacio de publicidad y barra inferior — **pero adaptado al dominio de quiniela** (pronósticos, ranking, jornadas), no a un simple marcador.

---

## 0. Reglas para Claude Code (leer primero)

- **La misión manda:** todo lo que se construya sirve a **administrar la quiniela**. Nunca implementar funcionalidad de "marcador" que no sirva al juego de pronósticos.
- **Mobile-first.** La referencia es una vista de teléfono (~390px de ancho lógico). Diseña primero para móvil; escala a tablet/desktop manteniendo una columna central de ancho máx. ~480–520px centrada.
- **Frontend 100% estático.** GitHub Pages no ejecuta backend. Toda lógica de escritura/validación/cálculo de puntos va a **Google Apps Script** desplegado como Web App (endpoint REST) o a un proxy serverless.
- **Nunca** expongas credenciales de cuenta de servicio, secretos de OAuth o claves de API con permisos de escritura en el cliente. El cliente solo usa el **ID de Cliente** público de Google Identity Services.
- **Paridad visual es criterio de aceptación** (estética de la captura), pero **subordinada a la misión**. Usa los tokens de diseño de la sección 3. Valida al final contra las listas de verificación (secciones 10 y 11).
- Código limpio, comentado en español, sin dependencias innecesarias. Preferir **React + Vite** o **JavaScript puro + Vite** (build estático). No usar nada que requiera servidor en runtime.
- **Todo el código, nombres de variables, funciones, componentes, pestañas y columnas de datos deben estar exclusivamente en español** (ver convención de nombres en cada sección).

---

## 1. Resumen del producto

Web app para **administrar y jugar una quiniela** de una liga de fútbol. Funciones núcleo:

- **Jugar pronósticos:** el participante ve los partidos de la **jornada** abierta y captura su pronóstico (marcador exacto y/o 1-X-2, según reglas) **antes de la fecha/hora límite**.
- **Cierre por fecha límite:** al llegar la hora de inicio del partido (o el cierre de jornada), los pronósticos se bloquean y ya no pueden editarse.
- **Resultados oficiales:** se capturan los marcadores reales de cada partido (ver sección 5 — fuente de datos vía API externo).
- **Cálculo de puntos:** el sistema asigna puntos según el reglamento (ej. acierto exacto vs. acierto de signo) y actualiza el marcador de cada participante.
- **Tabla de posiciones (ranking):** clasificación de participantes por puntos acumulados, por jornada y general.
- **Roles:** `jugador` (juega y consulta) y `admin` (crea jornadas/partidos, gestiona reglamento y participantes).

---

## 2. Stack técnico

| Capa | Tecnología |
|---|---|
| Frontend | React + Vite (build estático) **o** HTML/CSS/JS puro |
| Estilos | Variables CSS (tokens de diseño) + CSS modules/Tailwind opcional |
| Hosting | GitHub Pages (rama `gh-pages` o `/docs`) |
| Datos (lectura) | Google Sheets API v4 (lectura por token) o Apps Script GET |
| Datos (escritura + cálculo de puntos) | Google Apps Script Web App (POST con validación de token y de fecha límite) |
| Datos de partidos (en vivo, pasados y futuros) | Proveedor externo de API de resultados deportivos en vivo (ver sección 5) |
| Auth | Google Identity Services (OAuth 2.0) + alternativa por correo |
| CI/CD | GitHub Actions → build → deploy a Pages |

---

## 3. Sistema de diseño (Tokens de Diseño)

```css
:root {
  /* Fondos */
  --fondo-primario:        #0A0E14;  /* fondo general azul-negro muy oscuro */
  --fondo-elevado:         #0F1A24;  /* encabezado, barras de sección */
  --fondo-fila:            #0D1219;  /* filas (partidos, participantes) */
  --fondo-fila-hover:      #131c27;

  /* Texto */
  --texto-primario:        #FFFFFF;  /* equipos, títulos */
  --texto-secundario:      #8A94A6;  /* etiquetas (jornada, hora), subtítulos */
  --texto-atenuado:        #5A6473;

  /* Acentos */
  --acento-rojo:           #E2192B;  /* jornada activa, cierres de hoy, ítem nav activo */
  --acento-amarillo:       #F5C518;  /* encabezado "MI QUINIELA" */
  --acento-verde:          #2ECC71;  /* pronóstico acertado / puntos ganados (uso quiniela) */

  /* Estructura */
  --divisor: #1C2430;        /* líneas separadoras entre filas */
  --radio-sm: 4px;
  --radio-md: 8px;

  /* Espaciado / alturas */
  --espaciado-x: 16px;
  --alto-fila: 72px;
  --alto-encabezado: 56px;
  --alto-barra-fechas: 64px;
  --alto-nav-inferior: 64px;
  --alto-publicidad: 100px;
}
```

### Tipografía
- Familia: `system-ui, -apple-system, "Inter", "Roboto", sans-serif`.
- Título encabezado (nombre de la quiniela): 24px / bold.
- Nombres de equipos / participantes: 17–18px / 600 / blanco.
- Etiquetas (jornada, hora límite, estado): 12px / uppercase / `--texto-secundario` / letter-spacing 0.5px.
- Barra de fechas (jornadas): día/jornada 13px uppercase + fecha 15px, apilados.
- Puntos / contadores: 16px.

### Reglas de forma
- Filas: alto `--alto-fila`, padding lateral `--espaciado-x`, separador inferior `1px solid --divisor`.
- Escudos/íconos de equipo: ~40px a la izquierda.
- Insignia de estado: fondo `--acento-rojo` (cierra hoy) o `--acento-verde` (acertado), texto blanco bold, `border-radius: var(--radio-sm)`.

---

## 4. Estructura de pantalla (componentes)

Orden vertical (de arriba a abajo) en la pantalla principal **"Jugar"**:

1. **`<EncabezadoApp>`** — sticky top.
2. **`<BarraJornadas>`** — sticky bajo el encabezado (selector de jornada/fecha).
3. **`<BarraJornadaActual>`** — resumen de la jornada abierta.
4. **`<PanelMiQuiniela>`** (sección amarilla "MI QUINIELA").
5. **`<PartidosAPronosticar>`** (cuerpo scrollable: partidos a pronosticar).
6. **`<EspacioPublicidad>`** — fijo sobre la nav inferior.
7. **`<NavInferior>`** — fijo al fondo.

### 4.1 `<EncabezadoApp>` (siempre visible / sticky)
- **Izquierda:** `<SelectorQuiniela>` = ícono balón + **nombre de la quiniela/temporada** + flecha ▼. Si existe más de una quiniela/liga, permite cambiar la vista activa.
- **Derecha (íconos):**
  - Avisos/promos con punto rojo de notificación.
  - Búsqueda (lupa) — buscar partido, equipo o participante.
  - **Perfil/usuario con engrane** → estado de sesión. Si logueado: avatar + acceso a perfil; si no: abre `<ModalAutenticacion>`. Aquí el `admin` ve también su acceso al **panel de administración**.

### 4.2 `<BarraJornadas>` (selector de jornada/fecha)
- Tira horizontal de jornadas o fechas. La **jornada activa** se resalta en **rojo** + indicador inferior rojo.
- **Scroll horizontal** para ir a jornadas pasadas (resultados/puntos) o futuras (cuando abran).
- Cambiar de jornada → recarga sus partidos y el estado del jugador en esa jornada.
- Cada jornada muestra etiqueta (ej. "J12") y/o fecha (`DD.MM.`), con estado: abierta / cerrada / en juego / calificada.

### 4.3 `<BarraJornadaActual>`
- Ícono + texto "Jornada actual" (o "J12").
- Derecha: insignia roja con **partidos que cierran hoy** y contador de **pronósticos pendientes / total** (ej. `3/10`).

### 4.4 `<PanelMiQuiniela>` (sección amarilla "MI QUINIELA")
- Encabezado amarillo uppercase **"MI QUINIELA"**.
- Resumen del jugador: **posición en el ranking**, **puntos** (jornada y total), **pronósticos pendientes** de la jornada abierta y, si aplica, premio/racha.
- Acceso directo a "Ver ranking completo".

### 4.5 `<PartidosAPronosticar>` (partidos de la jornada)
- Encabezado uppercase gris (ej. "PARTIDOS — JORNADA 12").
- Lista de partidos de la jornada. Patrón de fila repetible (`<FilaPartido>`):
  - Escudo + **equipo local** vs **equipo visitante** + escudo.
  - **Hora límite / inicio del partido** y estado (abierto / cerrado / en juego / final).
  - **Entrada de pronóstico** (inputs de marcador y/o 1-X-2) cuando está **abierto**; **bloqueado** (solo lectura) cuando pasó la fecha límite.
  - Si ya hay resultado oficial: mostrar marcador real, el pronóstico del jugador y **puntos obtenidos** (resaltar en verde si acertó).
- Separador inferior sutil entre filas.

### 4.6 `<EspacioPublicidad>`
- Etiqueta pequeña "Publicidad".
- Banner ~ancho completo, alto `--alto-publicidad`, anclado sobre la nav inferior. **Espacio configurable** (imagen/iframe). No tapa contenido ni la nav.

### 4.7 `<NavInferior>` (fija) — 5 ítems adaptados a quiniela
1. **Jugar** — captura de pronósticos de la jornada abierta (activo en rojo por defecto).
2. **En Vivo** — partidos en curso y cómo van los pronósticos.
3. **Ranking** — tabla de posiciones de participantes (con insignia de cambios/posición).
4. **Avisos** — comunicados del organizador / noticias de la liga.
5. **Reglas** — reglamento de la quiniela (puntuación, fechas límite, premios).
- Ítem activo en rojo; inactivos en gris claro. Barra fija, no hace scroll.

### Comportamiento de layout
- Encabezado + BarraJornadas **sticky**. Cuerpo hace scroll por debajo.
- NavInferior + EspacioPublicidad **fijos**; reservar `padding-bottom: calc(var(--alto-publicidad) + var(--alto-nav-inferior))` para que el contenido nunca quede oculto.

---

## 5. Modelo de datos (Google Sheets)

Una hoja (Spreadsheet) con las siguientes pestañas:

- **`Quinielas`** — `id`, `nombre`, `temporada`, `estado` (`activa`|`cerrada`), `id_reglas_puntuacion`
- **`Equipos`** — `id`, `nombre`, `nombre_corto`, `url_escudo`
- **`Jornadas`** — `id`, `id_quiniela`, `numero`, `etiqueta` (ej. "J12"), `abre_en`, `cierra_en`, `estado` (`proxima`|`abierta`|`cerrada`|`calificada`)
- **`Partidos`** — `id`, `id_jornada`, `id_equipo_local`, `id_equipo_visitante`, `inicio_partido`, `fecha_limite` (fecha límite de pronóstico), `estado` (`programado`|`en_vivo`|`finalizado`), `goles_local`, `goles_visitante` (resultado oficial)
- **`Pronosticos`** — `id`, `id_usuario`, `id_partido`, `pronostico_local`, `pronostico_visitante`, `pronostico_resultado` (`1`|`X`|`2`), `enviado_en`, `bloqueado` (bool), `puntos_otorgados`
- **`ReglasPuntuacion`** — `id`, `puntos_marcador_exacto`, `puntos_resultado_correcto`, `puntos_diferencia_goles`, `notas`
- **`Posiciones`** (cacheable/derivable) — `id_quiniela`, `id_usuario`, `id_jornada`, `puntos_jornada`, `puntos_totales`, `posicion`
- **`Usuarios`** — `id_usuario`, `proveedor_auth` (`google`|`correo`), `rol` (`jugador`|`admin`), `telefono`, `telefono_verificado` (bool), `correo`, `nombre_visible`, `url_avatar`, `creado_en`
- **`Avisos`** — `id`, `id_quiniela`, `titulo`, `cuerpo`, `publicado_en`, `id_usuario_autor`
- **`Publicidad`** (opcional) — `espacio`, `url_imagen`, `url_destino`, `activo`
- **`RegistroAuditoria`** — `marca_tiempo`, `id_usuario_actor`, `accion`, `entidad`, `id_entidad`, `meta` (sin PII)

**Fuente de datos de partidos (en vivo, pasados y futuros):** La sección "En Vivo", así como el listado de partidos pasados y futuros de cada jornada (`PartidosAPronosticar`, `BarraJornadas`), **debe actualizarse únicamente mediante datos obtenidos de un proveedor externo de API de resultados deportivos en vivo** (p. ej. API-Football, SportRadar, o equivalente), y no mediante captura manual del `admin`. El backend (Apps Script o proxy serverless) debe consumir dicho API de forma periódica o mediante webhook, normalizar la respuesta al esquema de `Partidos` (`estado`, `goles_local`, `goles_visitante`, `inicio_partido`), y persistir/actualizar los registros en Google Sheets. La clave de API del proveedor debe mantenerse **únicamente del lado del servidor** (nunca en variables `VITE_*` ni expuesta al cliente), y el cálculo de puntos (`calificarJornada`) debe ejecutarse solo cuando el proveedor marque el partido como `finalizado`, manteniendo el principio de que el cliente nunca es fuente de verdad.

**Cálculos:** los puntos de cada `Pronostico` se calculan **server-side** comparando contra el resultado oficial de `Partidos` y el `ReglasPuntuacion` de la quiniela. `Posiciones` se recalcula al calificar una jornada (`Jornadas.estado = calificada`). Los contadores del UI (pendientes, cierran hoy) se derivan de `Pronosticos` + `Partidos.fecha_limite`.

---

## 6. Capa de acceso a datos

**Lectura (GET):**
- `obtenerJornadaActiva(idQuiniela)` — jornada abierta y sus partidos.
- `obtenerPartidosJornada(idJornada)` — partidos con su estado y fecha límite (sincronizados desde el proveedor de API externo, ver sección 5).
- `obtenerMisPronosticos(idJornada)` — pronósticos del usuario autenticado para esa jornada.
- `obtenerTablaPosiciones(idQuiniela, idJornada?)` — tabla de posiciones (general o por jornada).
- `obtenerAvisos(idQuiniela)` / `obtenerReglamento(idQuiniela)`.

**Escritura (POST → Apps Script con validación):**
- `enviarPronostico(idPartido, pronosticoLocal, pronosticoVisitante, pronosticoResultado)` — **rechaza si pasó `fecha_limite`** o si la jornada no está abierta; el `id_usuario` se deriva del token, no del cuerpo de la solicitud.
- `guardarUsuario(...)` — perfil del usuario autenticado.
- **(admin)** `crearJornada(...)`, `guardarPartido(...)`, `calificarJornada(idJornada)` (dispara cálculo de puntos y actualiza `Posiciones`; se ejecuta automáticamente cuando el proveedor de API marca el partido como `finalizado`), `publicarAviso(...)`.

**Estados UI:** `cargando` (skeleton), `vacio` ("Sin partidos en esta jornada"), `bloqueado` ("Pronóstico cerrado"), `error` (reintentar). Cachear lecturas por (quiniela + jornada).

---

## 7. Autenticación y roles (flujo)

1. **Disparador:** tap en perfil (encabezado) o intento de **enviar un pronóstico** sin sesión → abre `<ModalAutenticacion>`.
2. **Principal — Google:** botón "Continuar con Google" (Google Identity Services). Tras login, solicitar/asociar **número de celular** con verificación por **OTP/SMS**.
3. **Alternativa — Correo:** botón secundario "Continuar con correo" (enlace mágico o correo+contraseña vía Apps Script/serverless).
4. **Roles:** `jugador` por defecto; `admin` asignado manualmente en `Usuarios`. El panel de administración y los endpoints de admin **solo** son accesibles a `admin` (verificado server-side).
5. **Sesión:** reflejar avatar+nombre en el encabezado. Pronósticos, posición y pendientes ligados al `id_usuario`.
6. **Seguridad:** toda escritura (incluido enviar pronóstico) pasa por Apps Script validando el `id_token` de Google y la **fecha límite**. El cliente solo conoce el **ID de Cliente** público.

---

## 8. Estructura de carpetas sugerida

```
quiniela-liga/
├── CLAUDE.md                 # este archivo
├── index.html
├── package.json
├── vite.config.js
├── .github/
│   └── workflows/
│       └── deploy.yml        # build + deploy a GitHub Pages
├── public/
│   └── escudos/               # escudos de equipos
├── src/
│   ├── main.jsx
│   ├── App.jsx
│   ├── estilos/
│   │   └── tokens.css        # tokens de diseño (sección 3)
│   ├── componentes/
│   │   ├── EncabezadoApp.jsx
│   │   ├── SelectorQuiniela.jsx
│   │   ├── BarraJornadas.jsx
│   │   ├── BarraJornadaActual.jsx
│   │   ├── PanelMiQuiniela.jsx
│   │   ├── PartidosAPronosticar.jsx
│   │   ├── FilaPartido.jsx        # fila de partido con entrada de pronóstico
│   │   ├── TablaPosiciones.jsx    # ranking de participantes
│   │   ├── EspacioPublicidad.jsx
│   │   ├── NavInferior.jsx
│   │   ├── ModalAutenticacion.jsx
│   │   └── admin/
│   │       ├── PanelAdmin.jsx
│   │       ├── SincronizacionResultados.jsx  # sincronización de resultados desde el proveedor de API
│   │       └── GestorJornadas.jsx
│   ├── datos/
│   │   ├── api.js            # llamadas a Apps Script / Sheets API
│   │   └── auth.js           # Google Identity + alternativa correo + rol
│   └── hooks/
│       ├── usarJornada.js
│       ├── usarPronosticos.js
│       └── usarPosiciones.js
└── apps-script/
    └── Codigo.gs              # endpoint Web App: auth, fechas límite, sincronización con API de resultados, cálculo de puntos, posiciones
```

---

## 9. Variables de entorno / configuración

Crear `.env` (NO commitear) y exponer solo lo público vía Vite (prefijo `VITE_`):

```
VITE_GOOGLE_CLIENT_ID=xxxxxxxx.apps.googleusercontent.com
VITE_SHEETS_API_ENDPOINT=https://script.google.com/macros/s/XXX/exec
VITE_SHEET_ID=1AbC...        # solo si se lee con Sheets API directa
```

Secretos de escritura (cuenta de servicio, clave del proveedor de API de resultados en vivo, claves de SMS, etc.) viven **solo** dentro de Apps Script, nunca en el repo del frontend.

---

## 10. Build y despliegue a GitHub Pages

`vite.config.js` debe fijar `base` al nombre del repo:

```js
export default { base: '/quiniela-liga/' }
```

`.github/workflows/deploy.yml` (esquema):

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [ main ]
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-pages-artifact@v3
        with: { path: ./dist }
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: github-pages
    steps:
      - uses: actions/deploy-pages@v4
```

> Recordatorio: en Settings → Pages, fijar Source = GitHub Actions. El `ID de Cliente` de Google debe tener la URL de Pages como **origen autorizado** en la consola de Google Cloud.

---

## 11. Checklist de paridad visual y funcional (criterio de aceptación)

**Estética (de la captura):**
- [ ] Fondo azul-negro oscuro uniforme (`--fondo-primario`).
- [ ] Encabezado con `<SelectorQuiniela>` (balón + nombre quiniela + ▼), avisos con punto rojo, lupa, perfil con engrane.
- [ ] `BarraJornadas` con jornada activa en rojo + indicador inferior; scroll horizontal.
- [ ] Sección amarilla "MI QUINIELA".
- [ ] Lista de partidos con filas (escudos + equipos + hora/estado); separadores sutiles (`--divisor`).
- [ ] Espacio "Publicidad" anclado sobre la nav inferior.
- [ ] NavInferior de 5 ítems: Jugar (activo en rojo), En Vivo, Ranking, Avisos, Reglas.
- [ ] Encabezado y BarraJornadas sticky; NavInferior y EspacioPublicidad fijos; contenido nunca oculto.

**Misión (quiniela):**
- [ ] El jugador puede capturar pronósticos de la jornada abierta y se guardan.
- [ ] Los pronósticos se **bloquean automáticamente** al llegar la fecha límite (`fecha_limite`).
- [ ] Los resultados oficiales se sincronizan automáticamente desde el proveedor de API externo y disparan el cálculo de puntos.
- [ ] Los puntos se calculan server-side según `ReglasPuntuacion` y se reflejan en el ranking.
- [ ] `TablaPosiciones` muestra la tabla de posiciones (general y por jornada).
- [ ] Roles aplicados: solo `admin` accede a panel y endpoints de administración.

---

## 12. Seguridad — OWASP Top 10 (2021) con controles concretos

> **Nivel de datos:** se manejan **datos personales** (nombre, correo, **teléfono verificado**) y, además, **datos de integridad competitiva** (pronósticos, puntos, ranking) con potencial valor económico si hay premios. Tratar teléfono/correo como **PII** y los pronósticos/puntos como datos de **integridad crítica**.

### Principio rector de arquitectura
Con **GitHub Pages + Google Sheets**, el frontend es **público y no confiable**: cualquiera puede leer su código, sus variables `VITE_*` y llamar directamente al endpoint. La **única frontera de seguridad real es Apps Script (o el proxy serverless)**. Regla absoluta: **toda autorización, validación, control de fecha límite y cálculo de puntos ocurre del lado del servidor; el cliente nunca es fuente de verdad.**

### Riesgo específico de quiniela (integridad del juego) — máxima prioridad
- **Anti-trampa de fecha límite:** `enviarPronostico` debe **rechazar server-side** cualquier pronóstico recibido después de `fecha_limite` (usando la **hora del servidor**, nunca la del cliente). Un jugador no puede pronosticar un partido ya iniciado/terminado.
- **Inmutabilidad tras cierre:** una vez bloqueado (`bloqueado=true`) o calificado, el pronóstico **no puede modificarse**, ni siquiera por el mismo usuario.
- **No filtrar pronósticos ajenos antes del cierre:** los endpoints de lectura **no** deben exponer los pronósticos de otros participantes mientras la jornada esté abierta (evita copiar). Solo se revelan tras la fecha límite.
- **Resultados y cálculo de puntos solo server-side:** los resultados oficiales se reciben únicamente desde el proveedor de API verificado (sección 5) y `calificarJornada` es una operación interna del servidor; los puntos nunca se aceptan desde el cliente.
- **Idempotencia y auditoría:** calificar una jornada debe ser idempotente y quedar registrado en `RegistroAuditoria` (quién/qué proceso, cuándo, qué cambió).

### A01 — Broken Access Control
- El cliente **nunca** envía `id_usuario` confiable. El servidor deriva la identidad **del `id_token` verificado** y el **rol** desde `Usuarios`.
- Autorización por propiedad de recurso: un jugador solo lee/escribe **sus** `Pronosticos` y su `Usuarios`. Operaciones de admin (crear jornadas, publicar avisos, gestionar participantes) exigen `rol=admin` verificado server-side.
- **Denegar por defecto** (HTTP 403). No exponer endpoints de admin en la ruta pública sin verificación de rol.
- No confiar en ocultar el panel admin en el UI como medida de acceso (es solo cosmético).

### A02 — Cryptographic Failures (datos sensibles)
- **HTTPS obligatorio** extremo a extremo; nunca contenido mixto `http://`.
- **PII en reposo:** teléfono y correo normalizados y minimizados; considerar **hash** (SHA-256 con sal) del teléfono para búsquedas si el negocio no requiere el número en claro.
- **Tokens:** no persistir el `id_token` en `localStorage` (riesgo XSS). Mantener en memoria; renovar vía Google Identity. Sesión propia con cookie `HttpOnly`+`Secure`+`SameSite=Strict` si el proxy lo permite.
- No registrar PII ni tokens en logs ni en mensajes de error.

### A03 — Injection (incluye Formula/CSV Injection en Sheets)
- **CSV/Formula injection (crítico aquí):** todo valor de usuario escrito en celda que empiece con `=`, `+`, `-`, `@`, tab o CR debe **sanitizarse** (anteponer `'`) o rechazarse. Aplica a `nombre_visible`, correo y cualquier texto libre (avisos, notas).
- **Validación estricta server-side:** pronósticos numéricos no negativos y dentro de rango razonable; `pronostico_resultado` ∈ {`1`,`X`,`2`}; `id_partido`/`id_jornada` deben existir y pertenecer a la jornada abierta (lista blanca); correo válido; teléfono E.164.
- Renderizar datos de usuario con `textContent`, nunca `innerHTML`.

### A04 — Insecure Design
- **Rate limiting / anti-abuso** en el endpoint (por usuario/IP con `CacheService`/`PropertiesService`), y **especialmente** en envío de OTP/SMS (anti-bombing y control de costos).
- **Modelo de amenazas documentado:** anónimo en internet, jugador malicioso (pronosticar tarde, editar tras cierre, ver pronósticos ajenos, autoasignarse puntos/admin), abuso de SMS, scraping del ranking, manipulación de la sincronización con el proveedor de API de resultados.
- OTP de un solo uso, expirable (5–10 min), con límite de intentos y backoff.
- Lógica de fecha límite y cálculo de puntos **centralizada** en el servidor, no replicada/confiada en el cliente.

### A05 — Security Misconfiguration
- **CORS restrictivo:** Apps Script responde solo al **origen exacto** de GitHub Pages (no `*`).
- **CSP** en `index.html`: limitar `script-src`/`connect-src` a Google Identity, el endpoint de Apps Script y la red de publicidad; evitar `unsafe-inline`.
- Cabeceras: `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`.
- **Orígenes JavaScript autorizados** del ID de Cliente limitados a la URL de Pages (sin localhost en producción).
- Sin secretos en el repo: todo `VITE_*` es público por diseño.

### A06 — Vulnerable & Outdated Components
- Fijar versiones en `package.json`; `npm audit` en CI fallando ante vulnerabilidades altas/críticas. Habilitar **Dependabot**. Minimizar dependencias.
- **Espacio de publicidad = riesgo de terceros:** cargar la red publicitaria en `<iframe sandbox>`; nunca inyectar su script en el documento principal.

### A07 — Identification & Authentication Failures
- **Verificar el `id_token` de Google server-side SIEMPRE** (firma, `aud`=tu ID de Cliente, `iss`, `exp`). Nunca confiar en el payload decodificado en el cliente.
- **Teléfono verificado:** `telefono_verificado=true` solo tras validar OTP correcto en ventana; el cliente no puede establecer ese valor. Igual para `rol`.
- Alternativa por correo: enlace mágico de un solo uso/expirable, o contraseña con hashing fuerte (bcrypt/scrypt/argon2) — **nunca** PII/credenciales en claro en Sheets.
- Sesiones con expiración; logout invalida sesión server-side.

### A08 — Software & Data Integrity Failures
- Dependencias con **lockfile** (`npm ci`) en CI para builds reproducibles.
- Scripts de terceros (Google Identity, publicidad) desde orígenes oficiales; usar **SRI** donde la fuente sea estática.
- GitHub Actions con permisos mínimos (sección 10); acciones de terceros pinneadas por SHA en contextos sensibles.
- **Integridad del juego:** cualquier cambio a resultados oficiales o puntos queda en `RegistroAuditoria`; recalificar es idempotente y trazable.
- **Integridad de la sincronización de resultados:** validar la firma/origen de las respuestas del proveedor de API de resultados (o el secreto del webhook) antes de actualizar `Partidos`.

### A09 — Security Logging & Monitoring Failures
- Registrar en `RegistroAuditoria`/logs (sin PII): auth fallida, OTP fallido, accesos 403, intentos de pronóstico fuera de plazo, ediciones tras cierre, cambios de resultado/puntos, picos de rate limit, fallos de sincronización con el proveedor de API de resultados.
- Alertar ante anomalías (muchos OTP a un número/IP; ráfagas de pronósticos justo en la fecha límite; caída del proveedor de API de resultados).

### A10 — Server-Side Request Forgery (SSRF)
- Bajo en esta arquitectura, pero si el servidor hace fetch a URLs derivadas de input (`url_avatar`, `url_escudo`, `url_destino` de publicidad) o a la API del proveedor de resultados, **validar contra lista blanca de dominios** y no seguir redirecciones a rangos internos.

---

### Privacidad / cumplimiento (teléfono + PII + posible premio)
- **Minimización de datos** y justificación de por qué se pide el teléfono.
- **Consentimiento** explícito para SMS y aviso de privacidad accesible.
- **Derecho de borrado:** operación para eliminar la cuenta y su PII (conservando, si aplica, registros mínimos de integridad del juego de forma anonimizada).
- Si la quiniela maneja **dinero/premios**, revisar la legalidad local de apuestas/sorteos antes de operar; esto excede lo técnico.

---

### Checklist de seguridad (criterio de aceptación)
- [ ] **Fecha límite server-side:** se rechaza todo pronóstico posterior a `fecha_limite` usando hora del servidor.
- [ ] Pronósticos inmutables tras cierre/calificación; sin edición posterior.
- [ ] Pronósticos ajenos **no** visibles antes de la fecha límite.
- [ ] Resultados oficiales sincronizados **solo** desde el proveedor de API verificado; cálculo de puntos **solo** server-side; nunca aceptados del cliente.
- [ ] Identidad y **rol** derivados del **`id_token` verificado** (firma, `aud`, `iss`, `exp`); el cliente nunca envía `id_usuario`/`rol` confiables.
- [ ] Autorización por propiedad de recurso; denegar por defecto; endpoints admin protegidos por rol.
- [ ] **Sanitización anti-formula injection** en toda celda con input de usuario (`= + - @`).
- [ ] Validación server-side: pronóstico en rango, `resultado` ∈ {1,X,2}, ids en lista blanca de la jornada abierta, correo, teléfono E.164.
- [ ] CORS al origen de Pages; CSP en `index.html`; orígenes del ID de Cliente limitados.
- [ ] Rate limiting en endpoint y en OTP/SMS; OTP de un solo uso/expirable; `telefono_verificado` y `rol` solo server-side.
- [ ] Tokens fuera de `localStorage`; HTTPS estricto; sin contenido mixto.
- [ ] Publicidad en `<iframe sandbox>`; `npm ci` + `npm audit` en CI; Dependabot activo.
- [ ] `RegistroAuditoria` de eventos de integridad y seguridad sin PII; borrado de cuenta disponible.
- [ ] Sincronización con el proveedor de API de resultados autenticada/validada; clave de API solo en el servidor.

---

## 13. Orden de implementación sugerido

1. Scaffolding (Vite + tokens.css + layout con encabezado/BarraJornadas/nav inferior fijos).
2. Pantalla **Jugar** con **datos de prueba** (jornada, partidos, entrada de pronóstico) hasta lograr paridad visual y funcional (sección 11).
3. Apps Script + estructura de Sheets; lectura (`obtenerJornadaActiva`, `obtenerPartidosJornada`, `obtenerMisPronosticos`). **Aplicar desde aquí los controles de la sección 12** (validación, CORS, sanitización, **fecha límite server-side**).
4. Integración con el proveedor de API de resultados deportivos en vivo: sincronización periódica/webhook que actualiza `Partidos` (estado, marcador, horarios) — ver sección 5.
5. Auth (Google → alternativa correo) + verificación de teléfono por OTP + **roles**, con verificación de `id_token` server-side.
6. Envío de pronósticos con bloqueo por fecha límite; lectura del ranking (`obtenerTablaPosiciones`).
7. Calificación automática de jornadas: `calificarJornada` se dispara cuando el proveedor de API marca los partidos como `finalizado` (cálculo de puntos) + `RegistroAuditoria`. Panel **admin** para gestión de jornadas/participantes/avisos.
8. EspacioPublicidad en iframe sandbox; CI/CD a GitHub Pages (`npm ci` + `npm audit`); estados de carga/vacío/cerrado/error; pulido responsive.

---

### Definición de "hecho"
La web app **administra la quiniela**: el jugador autenticado (Google + teléfono verificado, o correo) captura pronósticos de la jornada abierta que se **bloquean automáticamente en la fecha límite** (validado server-side), los resultados oficiales se **sincronizan automáticamente desde un proveedor de API de resultados deportivos en vivo** y disparan el **cálculo de puntos** según el reglamento, el **ranking** de participantes se actualiza y se muestra, todo respetando los roles. La interfaz conserva la estética de la captura (sección 11) y la app pasa el **checklist de seguridad OWASP** (sección 12), desplegada en GitHub Pages bajo HTTPS con CORS y CSP correctos.
