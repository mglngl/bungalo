# CLAUDE.md — Quiniela Web App (Liga de Fútbol)

> Archivo principal de contexto para **Claude Code**.
> **Misión del producto:** construir una **web app** cuyo objetivo es **administrar el juego de quiniela de una liga de fútbol**: los usuarios **crean múltiples torneos privados o se unen a otros por invitación mediante un código único de 6 caracteres**, pronostican los resultados de las jornadas, el sistema **cierra la captura de pronósticos de cada partido 15 minutos antes de su inicio**, calcula puntos contra los resultados oficiales y mantiene una **tabla de posiciones** de participantes por torneo.
> **Arquitectura:** frontend estático en **GitHub Pages** servido como **aplicación de una sola página en un único archivo HTML (SPA estricta de archivo único)** + **Google Sheets como base de datos**. Autenticación vía **Google Auth** (principal, con número de celular verificado) y **correo electrónico** como alternativa.
> **Restricción de archivo único (NO negociable):** **todo el cliente reside en un solo `index.html`**; todas las vistas, paneles, componentes y flujos (autenticación, torneos, jornadas, pronósticos, ranking, estado de jugadores, configuración) se renderizan **dinámicamente dentro de ese mismo archivo**, mediante manipulación del DOM y **enrutamiento en memoria**, sin rutas separadas ni archivos HTML adicionales (ver sección 0 y §13). El backend en **Google Apps Script** es una pieza de servidor independiente y no forma parte de ese HTML.
> **Base visual (UI/UX):** se replica el estilo de la captura de referencia (interfaz deportiva móvil con su **propia identidad visual**) — encabezado con menú desplegable, navegación por fechas/jornadas, espacio de publicidad y barra inferior — **pero adaptado al dominio de quiniela** (pronósticos, ranking, jornadas), no a un simple marcador. La interfaz es **exclusivamente móvil** (en desktop conserva el radio de aspecto en un marco centrado), con **tema oscuro/claro** (oscuro por defecto) e **idiomas español/inglés** (español por defecto).

---

## 0. Reglas para Claude Code (leer primero)

- **La misión manda:** todo lo que se construya sirve a **administrar la quiniela**. Nunca implementar funcionalidad de "marcador" que no sirva al juego de pronósticos.
- **Arquitectura de un solo archivo HTML (SPA estricta) — restricción NO negociable.** **Todo el cliente del proyecto debe residir en un único archivo `index.html`.** Todos los componentes, vistas, paneles y flujos (autenticación, torneos, jornadas, pronósticos, ranking, estado de jugadores, configuración) se renderizan **dinámicamente dentro de ese mismo archivo**, respondiendo a la interacción del usuario, **sin generar rutas separadas ni archivos HTML adicionales**. El `index.html` es el **contenedor absoluto** del cliente: el marcado, los estilos (tokens de la sección 3 en un `<style>` embebido), la lógica de la app y los diccionarios de idioma (sección 8.1, como objetos JS embebidos) viven dentro de él. Toda navegación, transición entre vistas o carga de contenido se resuelve **en el cliente con JavaScript** (manipulación del DOM, componentes condicionales o **enrutamiento en memoria**), **nunca** cargando páginas externas. Esta restricción aplica **tanto al entorno de desarrollo como al artefacto final** desplegado en GitHub Pages. **Excepción:** el backend **Google Apps Script** (`Codigo.gs`) es una pieza de servidor que se despliega aparte en Google y **no** forma parte del HTML único; y los archivos **no-HTML** estrictamente necesarios por otras secciones (p. ej. `robots.txt`, `LICENSE`, configuración de build) pueden existir como archivos sueltos, ya que no son vistas de la aplicación.
- **Exclusivamente móvil (mobile-only).** La referencia es una vista de teléfono (~390–430px de ancho lógico). **El diseño es solo para móvil:** no se construyen vistas alternativas de tablet/desktop. Cuando la app se accede desde **desktop o pantallas grandes, se mantiene el radio de aspecto** del teléfono renderizando un **marco centrado** (ancho máx. `--ancho-app-max`, radio de aspecto `--aspecto-app`) sobre un fondo neutro (letterbox). Nunca se ensancha el layout a varias columnas ni se reflowea a escritorio.
- **Tema oscuro y claro.** La app soporta **dos temas: oscuro (por defecto) y claro**, conmutables por el usuario y persistentes entre sesiones. Todo color proviene de tokens (sección 3); el cambio de tema solo altera tokens de color, nunca la estructura.
- **Bilingüe (es/en).** La UI/UX soporta **exactamente dos idiomas: español (por defecto) e inglés** como alternativa, conmutables por el usuario y persistentes. Todo texto visible se obtiene de archivos de recursos de idioma (sección 8.1); no se escriben cadenas de texto "a mano" en los componentes.
- **Multi-torneo privado + unirse por invitación.** La app permite **crear múltiples juegos/torneos**, y todos son de carácter **privado**: **no existen torneos públicos ni son descubribles o listables**. El único modo de ingresar a un torneo es **por invitación**, compartiendo el **código único de 6 caracteres alfanuméricos** (solo letras y números) asignado al torneo. Un usuario participa en varios torneos a la vez (sección 5).
- **Frontend 100% estático y de archivo único.** GitHub Pages no ejecuta backend y **sirve un solo `index.html` autocontenido**. Toda lógica de escritura/validación/cálculo de puntos va a **Google Apps Script** desplegado como Web App (endpoint REST) o a un proxy serverless.
- **Nunca** expongas credenciales de cuenta de servicio, secretos de OAuth o claves de API con permisos de escritura en el cliente. El cliente solo usa el **ID de Cliente** público de Google Identity Services.
- **Paridad visual es criterio de aceptación** (estética de la captura), pero **subordinada a la misión**. Usa los tokens de diseño de la sección 3. Valida al final contra las listas de verificación (secciones 10 y 11).
- Código limpio, comentado en español, sin dependencias innecesarias. **JavaScript puro (Vanilla JS) embebido en el `index.html` es la vía preferida** para honrar la restricción de archivo único en desarrollo y producción. Si se usa **React**, debe autorizarse vía CDN embebido o **empaquetarse con Vite + `vite-plugin-singlefile`** de modo que el **build de producción produzca exactamente un `index.html` autocontenido** (sin chunks ni `assets/` externos). No usar nada que requiera servidor en runtime ni que rompa el principio de un solo archivo.
- **Todo el código, nombres de variables, funciones, componentes, pestañas y columnas de datos deben estar exclusivamente en español** (ver convención de nombres en cada sección). **Excepción:** el **texto visible de la UI** se externaliza en los archivos de idioma `es`/`en` (sección 8.1); los identificadores del código siguen siendo en español, pero las cadenas que ve el usuario viven en los recursos de traducción.

---

## 1. Resumen del producto

Web app para **administrar y jugar una quiniela** de una liga de fútbol. Funciones núcleo:

- **Crear y unirse a torneos (privados):** un usuario puede **crear múltiples juegos/torneos** (cada uno con su nombre, temporada y reglamento). **Todos los torneos son privados:** solo se puede ingresar **por invitación**, compartiendo el **código único de 6 caracteres alfanuméricos** asignado; no hay directorio público ni búsqueda de torneos ajenos. Para **unirse a un torneo de otra persona** se introduce dicho código. La vista activa siempre corresponde a uno de los torneos en los que participa.
- **Jugar pronósticos:** el participante ve los partidos de la **jornada** abierta y captura su pronóstico (marcador exacto y/o 1-X-2, según reglas), lo cual solo es posible **hasta 15 minutos antes del inicio de cada partido** (ese instante es la `fecha_limite` del partido).
- **Cierre por fecha límite:** la captura de cada partido se cierra **15 minutos antes de su hora de inicio** (`fecha_limite = inicio_partido − 15 min`); al llegar ese momento, el pronóstico de ese partido se bloquea y ya no puede editarse. El cierre es **por partido**, de forma independiente, no por jornada completa.
- **Resultados oficiales:** se capturan los marcadores reales de cada partido (ver sección 5 — fuente de datos vía API externo).
- **Cálculo de puntos:** el sistema asigna puntos según el reglamento (ej. acierto exacto vs. acierto de signo) y actualiza el marcador de cada participante.
- **Tabla de posiciones (ranking):** clasificación de participantes por puntos acumulados, por jornada y general, **por torneo**.
- **Estado de los jugadores (por torneo):** cada participante recibe un **estado de cumplimiento de pronósticos** en tres categorías —**Atrasado**, **En Riesgo** o **Al día**—, calculado server-side a partir de sus partidos sin pronosticar y las fechas límite de cada partido (definición en §4.4.1 y §5).
- **Roles:** `jugador` (juega y consulta) y `admin`. El rol se evalúa **por torneo**: quien **crea** un torneo es su **admin** (gestiona jornadas/partidos, reglamento y participantes de **ese** torneo); en torneos a los que solo se unió, es `jugador`.

---

## 2. Stack técnico

| Capa | Tecnología |
|---|---|
| Frontend | **Un único `index.html` (SPA estricta de archivo único).** Vanilla JS embebido (preferido) **o** React vía CDN / `vite-plugin-singlefile`; todo el marcado, estilos y lógica viven en el mismo archivo |
| Estilos | Variables CSS (tokens de diseño) en un **`<style>` embebido** en el `index.html` + **temas oscuro/claro** vía `data-tema` (Tailwind solo vía CDN si se desea; sin CSS modules) |
| Internacionalización | Diccionarios `es`/`en` como **objetos JS embebidos** en el `index.html` + utilidad propia ligera; idioma persistido |
| Marco móvil | Layout de ancho fijo (`--ancho-app-max`) con radio de aspecto en desktop (sin reflow a escritorio) |
| Hosting | GitHub Pages sirviendo **un único `index.html` autocontenido** (rama `gh-pages` o `/docs`) |
| Datos (lectura) | Google Sheets API v4 (lectura por token) o Apps Script GET |
| Datos (escritura + cálculo de puntos) | Google Apps Script Web App (POST con validación de token y de fecha límite) |
| Datos de partidos (en vivo, pasados y futuros) | Proveedor externo de API de resultados deportivos en vivo (ver sección 5) |
| Auth | Google Identity Services (OAuth 2.0) + alternativa por correo |
| CI/CD | GitHub Actions → **build single-file** (Vite + `vite-plugin-singlefile`, **sin sourcemaps**, minificado) → deploy a Pages |

---

## 3. Sistema de diseño (Tokens de Diseño)

```css
/* ===========================================================
   TOKENS DE DISEÑO
   Tema OSCURO por defecto. El tema se controla con el atributo
   data-tema en <html> ("oscuro" | "claro"). El tema CLARO solo
   sobrescribe tokens de color; la ESTRUCTURA no cambia con el tema.
   =========================================================== */

:root,
[data-tema="oscuro"] {
  /* Fondos */
  --fondo-primario:        #0A0E14;  /* fondo general azul-negro muy oscuro */
  --fondo-elevado:         #0F1A24;  /* encabezado, barras de sección */
  --fondo-fila:            #0D1219;  /* filas (partidos, participantes) */
  --fondo-fila-hover:      #131c27;
  --fondo-letterbox:       #05070A;  /* fondo neutro detrás del marco en desktop */

  /* Texto */
  --texto-primario:        #FFFFFF;  /* equipos, títulos */
  --texto-secundario:      #8A94A6;  /* etiquetas (jornada, hora), subtítulos */
  --texto-atenuado:        #5A6473;

  /* Acentos */
  --acento-rojo:           #E2192B;  /* jornada activa, cierres de hoy, ítem nav activo */
  --acento-amarillo:       #F5C518;  /* encabezado "MI QUINIELA" */
  --acento-verde:          #2ECC71;  /* pronóstico acertado / puntos ganados (uso quiniela) */

  /* Estructura (COMPARTIDA entre temas) */
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

  /* Marco de aplicación (vista móvil exclusiva) */
  --ancho-app-max: 430px;    /* ancho lógico máx. del marco tipo teléfono */
  --aspecto-app: 9 / 19.5;   /* radio de aspecto del marco en pantallas grandes */
}

/* === Tema CLARO — solo color, hereda la estructura del bloque anterior === */
[data-tema="claro"] {
  --fondo-primario:        #F4F6F9;  /* fondo general claro */
  --fondo-elevado:         #FFFFFF;  /* encabezado, barras de sección */
  --fondo-fila:            #FFFFFF;  /* filas */
  --fondo-fila-hover:      #EDF1F6;
  --fondo-letterbox:       #DCE2EA;  /* fondo neutro detrás del marco en desktop */

  --texto-primario:        #0A0E14;
  --texto-secundario:      #5A6473;
  --texto-atenuado:        #8A94A6;

  --acento-rojo:           #D11425;  /* ajustado para contraste sobre claro */
  --acento-amarillo:       #B8860B;  /* amarillo legible sobre fondo claro */
  --acento-verde:          #1E9E5A;

  --divisor:               #E2E7EE;
}
```

> **Tema por defecto:** oscuro. La preferencia del usuario se aplica fijando `data-tema` en `<html>` y se persiste (ver sección 8.2). Respetar `prefers-color-scheme` solo como valor inicial si el usuario aún no eligió; una vez elegido, manda su preferencia.

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
- **Izquierda:** `<SelectorQuiniela>` = ícono balón + **nombre del torneo/temporada activo** + flecha ▼. Al abrirlo despliega:
  - La **lista de torneos** en los que el usuario participa (cambiar la vista activa).
  - Acción **"Crear torneo nuevo"** → abre `<ModalCrearQuiniela>` (genera y muestra el **código de 6 caracteres** para invitar).
  - Acción **"Unirse con código"** → abre `<ModalUnirseQuiniela>` (captura el código de 6 caracteres alfanuméricos).
- **Derecha (íconos):**
  - Avisos/promos con punto rojo de notificación.
  - Búsqueda (lupa) — buscar partido, equipo o participante.
  - **Perfil/usuario con engrane** → estado de sesión. Si logueado: avatar + acceso a perfil; si no: abre `<ModalAutenticacion>`. Aquí el `admin` (del torneo activo) ve también su acceso al **panel de administración**. Desde aquí se accede a **preferencias**: `<SelectorTema>` (oscuro/claro) y `<SelectorIdioma>` (es/en).
- **Conmutadores rápidos de tema e idioma:** además del menú de perfil, exponer accesos rápidos (íconos de sol/luna para tema y `ES/EN` para idioma) accesibles con una mano. Ambos persisten la elección y aplican el cambio al instante (sin recargar).

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
- **Insignia de estado del jugador** (Atrasado / En Riesgo / Al día) del usuario en el torneo activo — ver §4.4.1.
- Acceso directo a "Ver ranking completo".

### 4.4.1 `<EstadoJugadores>` — Estado de cumplimiento de pronósticos (por torneo)
Clasifica a cada participante del torneo activo en **tres categorías mutuamente excluyentes**, según sus **partidos sin pronosticar** y la `fecha_limite` de cada partido (= `inicio_partido` − 15 min), evaluadas con la **hora del servidor**:

- **Atrasado** (insignia color `--acento-rojo`) — tiene **al menos un partido sin pronosticar que ya está bloqueado** (su `fecha_limite` ya pasó); es decir, acumula pronósticos perdidos.
- **En Riesgo** (insignia color `--acento-amarillo`) — está **al día** (sin pronósticos perdidos), pero tiene **al menos un partido sin pronosticar que se bloquea en menos de 24 horas**.
- **Al día** (insignia color `--acento-verde`) — **sin pronósticos perdidos** y **ningún** partido pendiente se bloquea en las **próximas 24 horas**.

**Orden de evaluación (prioridad):** se comprueba primero **Atrasado**; si no aplica, **En Riesgo**; si no, **Al día**.

**Dónde se muestra:**
- En `<PanelMiQuiniela>`: insignia con el **estado propio** del usuario en el torneo activo.
- En el **Ranking** (`<TablaPosiciones>`, ítem *Ranking* de la nav inferior): insignia de estado **por cada participante**, junto a su posición y puntos.
- La insignia solo expone la **categoría** de estado; **nunca** el contenido de los pronósticos ajenos (respeta §12: no filtrar pronósticos antes del cierre).

**Cálculo:** se deriva **server-side** de `Pronosticos` + `Partidos.fecha_limite` contra la **hora del servidor**. Como las categorías dependen de una ventana de **24 h relativa al momento de consulta**, el estado se calcula al leer (o se cachea con TTL corto); no se congela en `Posiciones`.

### 4.5 `<PartidosAPronosticar>` (partidos de la jornada)
- Encabezado uppercase gris (ej. "PARTIDOS — JORNADA 12").
- Lista de partidos de la jornada. Patrón de fila repetible (`<FilaPartido>`):
  - Escudo + **equipo local** vs **equipo visitante** + escudo.
  - **Hora límite** (15 minutos antes del inicio) e **inicio del partido**, con estado (abierto / cerrado / en juego / final); mostrar cuenta regresiva al cierre cuando falte poco.
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

### 4.8 Componentes superpuestos (modales y selectores)
- **`<ModalCrearQuiniela>`** — formulario para crear torneo (nombre, temporada, reglamento base). Al confirmar, el **servidor genera el código único de 6 caracteres** y el modal lo muestra con opción de copiar/compartir.
- **`<ModalUnirseQuiniela>`** — un solo campo para el **código de 6 caracteres** (mayúsculas/minúsculas y dígitos). Valida formato en cliente (`^[A-Za-z0-9]{6}$`) y delega la verificación real al servidor; muestra error si el código no existe o ya pertenece al torneo.
- **`<SelectorTema>`** — alterna oscuro/claro; refleja el estado actual y persiste la elección.
- **`<SelectorIdioma>`** — alterna español/inglés; refleja el idioma activo y persiste la elección.

### Comportamiento de layout
- **Renderizado dentro del archivo único.** Todos los componentes descritos arriba, los modales/selectores (§4.8) y las cinco vistas de la nav inferior (§4.7) se montan y desmontan **dinámicamente dentro del mismo `index.html`** mediante manipulación del DOM / componentes condicionales. El cambio entre *Jugar / En Vivo / Ranking / Avisos / Reglas*, la apertura de modales y el cambio de torneo activo son **transiciones en memoria** (enrutamiento en memoria); **nunca** navegan a otra página ni cargan archivos HTML adicionales. Este apartado describe el **mecanismo**; el diseño visual (secciones 3 y 4) se conserva sin cambios.
- **Vista móvil exclusiva.** El layout se diseña para una sola columna de teléfono. **No** hay breakpoints que reorganicen la interfaz para escritorio.
- **Marco con radio de aspecto en desktop/pantallas grandes.** Por encima de `--ancho-app-max`, la app se renderiza dentro de un **marco centrado** que conserva el **radio de aspecto** del teléfono (`--aspecto-app`), con el resto de la ventana en `--fondo-letterbox`. La interfaz interna **no cambia**: es la misma vista móvil, centrada y enmarcada.
- Encabezado + BarraJornadas **sticky** (dentro del marco). Cuerpo hace scroll por debajo.
- NavInferior + EspacioPublicidad **fijos** (anclados al marco, no a la ventana); reservar `padding-bottom: calc(var(--alto-publicidad) + var(--alto-nav-inferior))` para que el contenido nunca quede oculto.

---

## 5. Modelo de datos (Google Sheets)

Una hoja (Spreadsheet) con las siguientes pestañas:

- **`Quinielas`** — `id`, `nombre`, `temporada`, `codigo` (6 caracteres alfanuméricos, **único**, para invitar/unirse), `visibilidad` (**siempre `privada`**: el torneo solo admite ingreso por invitación con `codigo`; no existe valor `publica`), `id_usuario_creador`, `estado` (`activa`|`cerrada`), `id_reglas_puntuacion`, `creado_en`
- **`Participaciones`** — `id`, `id_quiniela`, `id_usuario`, `rol_en_quiniela` (`admin`|`jugador`; el creador entra como `admin`), `unido_en`. **Tabla puente** que permite que un usuario pertenezca a **varios torneos** y define su rol **por torneo**. Clave lógica única (`id_quiniela` + `id_usuario`) para evitar duplicados.
- **`Equipos`** — `id`, `nombre`, `nombre_corto`, `url_escudo`
- **`Jornadas`** — `id`, `id_quiniela`, `numero`, `etiqueta` (ej. "J12"), `abre_en`, `cierra_en`, `estado` (`proxima`|`abierta`|`cerrada`|`calificada`)
- **`Partidos`** — `id`, `id_jornada`, `id_equipo_local`, `id_equipo_visitante`, `inicio_partido`, `fecha_limite` (**fecha/hora límite de pronóstico = `inicio_partido` − 15 minutos**, calculada server-side a partir del `inicio_partido` oficial del proveedor de API), `estado` (`programado`|`en_vivo`|`finalizado`), `goles_local`, `goles_visitante` (resultado oficial)
- **`Pronosticos`** — `id`, `id_usuario`, `id_partido`, `pronostico_local`, `pronostico_visitante`, `pronostico_resultado` (`1`|`X`|`2`), `enviado_en`, `bloqueado` (bool), `puntos_otorgados`
- **`ReglasPuntuacion`** — `id`, `puntos_marcador_exacto`, `puntos_resultado_correcto`, `puntos_diferencia_goles`, `notas`
- **`Posiciones`** (cacheable/derivable) — `id_quiniela`, `id_usuario`, `id_jornada`, `puntos_jornada`, `puntos_totales`, `posicion`, `estado_jugador` (`atrasado`|`en_riesgo`|`al_dia`, **derivado server-side**; ver §4.4.1)
- **`Usuarios`** — `id_usuario`, `proveedor_auth` (`google`|`correo`), `rol` (`jugador`|`admin`; rol **global/base** — el rol operativo se evalúa **por torneo** en `Participaciones`), `telefono`, `telefono_verificado` (bool), `correo`, `nombre_visible`, `url_avatar`, `idioma_preferido` (`es`|`en`, por defecto `es`), `tema_preferido` (`oscuro`|`claro`, por defecto `oscuro`), `creado_en`
- **`Avisos`** — `id`, `id_quiniela`, `titulo`, `cuerpo`, `publicado_en`, `id_usuario_autor`
- **`Publicidad`** (opcional) — `espacio`, `url_imagen`, `url_destino`, `activo`
- **`RegistroAuditoria`** — `marca_tiempo`, `id_usuario_actor`, `accion`, `entidad`, `id_entidad`, `meta` (sin PII)

**Fuente de datos de partidos (en vivo, pasados y futuros):** La sección "En Vivo", así como el listado de partidos pasados y futuros de cada jornada (`PartidosAPronosticar`, `BarraJornadas`), **debe actualizarse únicamente mediante datos obtenidos de un proveedor externo de API de resultados deportivos en vivo** (p. ej. API-Football, SportRadar, o equivalente), y no mediante captura manual del `admin`. El backend (Apps Script o proxy serverless) debe consumir dicho API de forma periódica o mediante webhook, normalizar la respuesta al esquema de `Partidos` (`estado`, `goles_local`, `goles_visitante`, `inicio_partido`), y persistir/actualizar los registros en Google Sheets. La clave de API del proveedor debe mantenerse **únicamente del lado del servidor** (nunca en variables `VITE_*` ni expuesta al cliente), y el cálculo de puntos (`calificarJornada`) debe ejecutarse solo cuando el proveedor marque el partido como `finalizado`, manteniendo el principio de que el cliente nunca es fuente de verdad.

**Cálculos:** los puntos de cada `Pronostico` se calculan **server-side** comparando contra el resultado oficial de `Partidos` y el `ReglasPuntuacion` de la quiniela. `Posiciones` se recalcula al calificar una jornada (`Jornadas.estado = calificada`). Los contadores del UI (pendientes, cierran hoy) se derivan de `Pronosticos` + `Partidos.fecha_limite`. El **estado de jugador** (`atrasado`|`en_riesgo`|`al_dia`, §4.4.1) también se deriva de `Pronosticos` + `Partidos.fecha_limite` (= `inicio_partido` − 15 min) contra la **hora del servidor**; al depender de una ventana de 24 h relativa al momento de consulta, se calcula al leer o se cachea con TTL corto.

---

## 6. Capa de acceso a datos

**Lectura (GET):**
- `obtenerMisQuinielas()` — torneos en los que participa el usuario autenticado (desde `Participaciones`), con su rol por torneo.
- `obtenerJornadaActiva(idQuiniela)` — jornada abierta y sus partidos.
- `obtenerPartidosJornada(idJornada)` — partidos con su estado y fecha límite (sincronizados desde el proveedor de API externo, ver sección 5).
- `obtenerMisPronosticos(idJornada)` — pronósticos del usuario autenticado para esa jornada.
- `obtenerTablaPosiciones(idQuiniela, idJornada?)` — tabla de posiciones (general o por jornada), incluyendo el **`estado_jugador`** (`atrasado`|`en_riesgo`|`al_dia`, §4.4.1) de cada participante, calculado server-side.
- `obtenerEstadoJugadores(idQuiniela)` — estado de cumplimiento (`atrasado`|`en_riesgo`|`al_dia`) de cada participante del torneo activo, derivado server-side de los partidos sin pronosticar y la `fecha_limite` de cada partido (§4.4.1).
- `obtenerAvisos(idQuiniela)` / `obtenerReglamento(idQuiniela)`.

**Escritura (POST → Apps Script con validación):**
- `crearQuiniela(nombre, temporada, reglas)` — crea un torneo; el **servidor genera el `codigo` único de 6 caracteres alfanuméricos** (con verificación de colisión) y registra al creador en `Participaciones` con `rol_en_quiniela=admin`. Devuelve el código.
- `unirseQuiniela(codigo)` — valida el `codigo` server-side; si existe y el usuario no participa aún, crea la `Participacion` con `rol_en_quiniela=jugador`. Rechaza códigos inválidos/inexistentes y la doble unión.
- `salirQuiniela(idQuiniela)` — elimina la participación del usuario (no permitido para el creador salvo transferencia/baja del torneo).
- `enviarPronostico(idPartido, pronosticoLocal, pronosticoVisitante, pronosticoResultado)` — **rechaza si ya pasó la `fecha_limite` del partido** (es decir, si faltan **menos de 15 minutos** para el inicio o el partido ya comenzó) o si la jornada no está abierta; el `id_usuario` se deriva del token, no del cuerpo de la solicitud. Verifica además que el usuario **participe** en el torneo del partido.
- `guardarUsuario(...)` — perfil del usuario autenticado, incluidas sus preferencias `idioma_preferido` (`es`|`en`) y `tema_preferido` (`oscuro`|`claro`).
- **(admin del torneo)** `crearJornada(...)`, `guardarPartido(...)`, `calificarJornada(idJornada)` (dispara cálculo de puntos y actualiza `Posiciones`; se ejecuta automáticamente cuando el proveedor de API marca el partido como `finalizado`), `publicarAviso(...)`. Estas operaciones exigen `rol_en_quiniela=admin` **en ese torneo**, verificado server-side.

**Estados UI:** `cargando` (skeleton), `vacio` ("Sin partidos en esta jornada"), `bloqueado` ("Pronóstico cerrado"), `error` (reintentar). Cachear lecturas por (quiniela + jornada).

---

## 7. Autenticación y roles (flujo)

1. **Disparador:** tap en perfil (encabezado) o intento de **enviar un pronóstico** sin sesión → abre `<ModalAutenticacion>`.
2. **Principal — Google:** botón "Continuar con Google" (Google Identity Services). Tras login, solicitar/asociar **número de celular** con verificación por **OTP/SMS**.
3. **Alternativa — Correo:** botón secundario "Continuar con correo" (enlace mágico o correo+contraseña vía Apps Script/serverless).
4. **Roles (por torneo):** todo usuario es `jugador` por defecto. El **creador** de un torneo queda como `admin` de **ese** torneo (registrado en `Participaciones.rol_en_quiniela`). El panel de administración y los endpoints de admin de un torneo **solo** son accesibles a quien tenga `rol_en_quiniela=admin` en él (verificado server-side); ser admin de un torneo no otorga privilegios en otros.
5. **Sesión:** reflejar avatar+nombre en el encabezado. Pronósticos, posición y pendientes ligados al `id_usuario`.
6. **Seguridad:** toda escritura (incluido enviar pronóstico) pasa por Apps Script validando el `id_token` de Google y la **fecha límite**. El cliente solo conoce el **ID de Cliente** público.

---

## 8. Estructura de carpetas sugerida

```
quiniela-liga/
├── CLAUDE.md                 # este archivo
├── index.html                # ★ ÚNICO ARCHIVO DEL CLIENTE (SPA estricta): contiene TODO
│                             #   <head>: meta TDM opt-out (tdm-reservation) + robots noai/noindex
│                             #           + aviso de copyright + CSP (secciones 12 y 13)
│                             #   <style>: tokens de diseño + temas oscuro/claro (sección 3, embebidos)
│                             #   <body>: marco móvil + nodo raíz de montaje (p. ej. <div id="app">)
│                             #   <script>: TODA la lógica (vistas, modales, i18n, datos, auth, estado)
│
│   ── Organización LÓGICA interna del <script> (NO son archivos: son módulos/secciones en memoria) ──
│      • Tema:               ProveedorTema, usarTema  (oscuro|claro; aplica data-tema)
│      • Idioma (i18n §8.1): RECURSOS.es / RECURSOS.en (objetos embebidos), t(), ProveedorIdioma, usarIdioma
│      • Vistas/Componentes: EncabezadoApp, SelectorQuiniela (lista + crear + unirse), BarraJornadas,
│                            BarraJornadaActual, PanelMiQuiniela, PartidosAPronosticar, FilaPartido,
│                            TablaPosiciones, EstadoJugadores (§4.4.1), EspacioPublicidad, NavInferior
│      • Modales/Selectores: ModalAutenticacion, ModalCrearQuiniela (muestra el código generado),
│                            ModalUnirseQuiniela (código de 6 caracteres), SelectorTema, SelectorIdioma
│      • Admin (por torneo):  PanelAdmin, SincronizacionResultados, GestorJornadas
│      • Datos:              api (llamadas a Apps Script / Sheets), auth (Google Identity + correo + rol)
│      • Hooks/estado:       usarMisQuinielas, usarJornada, usarPronosticos, usarPosiciones,
│                            usarEstadoJugadores (§4.4.1)
│      • Enrutador en memoria: conmuta Jugar / En Vivo / Ranking / Avisos / Reglas SIN rutas ni HTML extra
│
│   (Los escudos provienen del proveedor de API vía `url_escudo`; si se requiere alguno local,
│    se incrusta como data URI dentro del index.html para no romper el archivo único.)
│
├── LICENSE                   # licencia restrictiva — Todos los derechos reservados (sección 13)
├── package.json
├── vite.config.js            # build single-file: inline total + minify + SIN sourcemaps + sin comentarios
│                             #   (secciones 10 y 13)
├── .env                      # NO commitear (sección 9)
├── .github/
│   └── workflows/
│       └── deploy.yml        # build single-file + deploy a GitHub Pages
├── public/                   # assets servidos en la RAÍZ del sitio (no son vistas HTML)
│   └── robots.txt            # Disallow a crawlers no autorizados + noai + noindex (sección 13)
└── apps-script/
    └── Codigo.gs             # BACKEND aparte (Google Apps Script) — NO forma parte del index.html:
                              #   auth, fechas límite, sincronización con API de resultados,
                              #   cálculo de puntos, posiciones, generación de códigos de torneo
```

### 8.1 Recursos de idioma (i18n)
- Dos diccionarios planos como **objetos JS embebidos** en el `index.html`: `RECURSOS.es` (por defecto) y `RECURSOS.en`, con **las mismas claves**. Toda cadena visible vive aquí; los componentes/vistas solo referencian claves vía `t('clave')`. **No se permite texto incrustado** en el marcado generado. (El requisito de externalizar el texto se mantiene; solo cambia el soporte: objetos en memoria en lugar de archivos `.json` sueltos, para honrar el archivo único.)
- Mantener `RECURSOS.es` como fuente de verdad; cada clave nueva debe existir en ambos objetos (validar paridad de claves en build/CI si es viable).

### 8.2 Persistencia de preferencias (tema e idioma)
- **Tema** e **idioma** se guardan en `localStorage` (`tema` = `oscuro|claro`, `idioma` = `es|en`). Estas **no son credenciales**, por lo que `localStorage` es aceptable aquí (a diferencia de los tokens, sección 12).
- Al iniciar: leer preferencia guardada → si no existe, usar **oscuro** y **es** por defecto (idioma puede inicializarse desde `navigator.language` solo si es `en`, pero el valor por defecto del producto es `es`).
- Si el usuario está autenticado, sincronizar la preferencia con `Usuarios.idioma_preferido`/`tema_preferido` para portabilidad entre dispositivos.

---

## 9. Variables de entorno / configuración

Crear `.env` (NO commitear) y exponer solo lo público vía Vite (prefijo `VITE_`):

```
VITE_GOOGLE_CLIENT_ID=xxxxxxxx.apps.googleusercontent.com
VITE_SHEETS_API_ENDPOINT=https://script.google.com/macros/s/XXX/exec
VITE_SHEET_ID=1AbC...        # solo si se lee con Sheets API directa
```

> En el build single-file, estas variables `VITE_*` quedan **incrustadas dentro del `index.html`** resultante y son **públicas por diseño** (cualquiera puede leerlas en el HTML servido). **Nunca** colocar secretos aquí.

Secretos de escritura (cuenta de servicio, clave del proveedor de API de resultados en vivo, claves de SMS, etc.) viven **solo** dentro de Apps Script, nunca en el repo del frontend.

---

## 10. Build y despliegue a GitHub Pages

`vite.config.js` debe fijar `base` al nombre del repo **y producir un único `index.html` autocontenido** (inline total, sin sourcemaps, minificado y sin comentarios — ver sección 13):

```js
import { defineConfig } from 'vite'
import { viteSingleFile } from 'vite-plugin-singlefile'

export default defineConfig({
  base: '/quiniela-liga/',
  plugins: [viteSingleFile()],          // inlina JS + CSS en un único index.html
  build: {
    target: 'esnext',
    sourcemap: false,                    // PI: SIN source maps en producción (§13)
    cssCodeSplit: false,                 // un solo bundle de estilos, embebido
    assetsInlineLimit: 100000000,        // fuerza inlining de assets dentro del HTML
    minify: 'terser',
    terserOptions: {
      format: { comments: false },       // suprime comentarios del bundle
      mangle: true,                      // nombres no descriptivos (menos legible)
      compress: { drop_console: true, drop_debugger: true }
    },
    rollupOptions: { output: { inlineDynamicImports: true } } // sin chunks externos
  }
})
```

> El artefacto del build debe ser **un solo `index.html` autocontenido**: sin carpeta `assets/`, sin archivos `.js`/`.css` sueltos y **sin archivos `.map`**. `robots.txt` y `LICENSE` se sirven aparte (no son vistas).

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
- [ ] Fondo azul-negro oscuro uniforme (`--fondo-primario`) en el **tema oscuro** (por defecto).
- [ ] Encabezado con `<SelectorQuiniela>` (balón + nombre torneo + ▼), avisos con punto rojo, lupa, perfil con engrane.
- [ ] `BarraJornadas` con jornada activa en rojo + indicador inferior; scroll horizontal.
- [ ] Sección amarilla "MI QUINIELA".
- [ ] Lista de partidos con filas (escudos + equipos + hora/estado); separadores sutiles (`--divisor`).
- [ ] Espacio "Publicidad" anclado sobre la nav inferior.
- [ ] NavInferior de 5 ítems: Jugar (activo en rojo), En Vivo, Ranking, Avisos, Reglas.
- [ ] Encabezado y BarraJornadas sticky; NavInferior y EspacioPublicidad fijos; contenido nunca oculto.

**Plataforma y preferencias:**
- [ ] **SPA de archivo único:** el cliente desplegado es **un solo `index.html` autocontenido**; toda navegación entre vistas (Jugar / En Vivo / Ranking / Avisos / Reglas), modales y cambio de torneo ocurre **en memoria**, sin rutas ni archivos HTML adicionales.
- [ ] **Vista móvil exclusiva:** una sola columna de teléfono, sin reflow a escritorio.
- [ ] En **desktop/pantallas grandes** la app se muestra en un **marco centrado que conserva el radio de aspecto** (`--aspecto-app`), con letterbox; la UI interna no cambia.
- [ ] **Tema oscuro/claro funcional**, **oscuro por defecto**, conmutable y **persistente**; el cambio solo afecta color, no estructura.
- [ ] **Idioma es/en conmutable**, **español por defecto**, **persistente**, y **todo** el texto visible proviene de `i18n` (sin cadenas incrustadas).

**Misión (quiniela):**
- [ ] El usuario puede **crear torneos** (se genera código único de 6 caracteres alfanuméricos) y **unirse a otros** mediante código; **todos los torneos son privados** (ingreso solo por invitación con el código, sin descubrimiento ni listado público).
- [ ] El usuario participa en **múltiples torneos** y puede cambiar el torneo activo.
- [ ] El jugador puede capturar pronósticos de la jornada abierta y se guardan.
- [ ] Los pronósticos se **bloquean automáticamente** al llegar la fecha límite de cada partido (`fecha_limite = inicio_partido − 15 min`), de forma **independiente por partido**.
- [ ] Los resultados oficiales se sincronizan automáticamente desde el proveedor de API externo y disparan el cálculo de puntos.
- [ ] Los puntos se calculan server-side según `ReglasPuntuacion` y se reflejan en el ranking.
- [ ] `TablaPosiciones` muestra la tabla de posiciones (general y por jornada) **del torneo activo**.
- [ ] Se muestra el **estado de cada jugador** —**Atrasado**, **En Riesgo** o **Al día**— en `PanelMiQuiniela` (propio) y en `TablaPosiciones` (todos), calculado server-side según §4.4.1.
- [ ] Roles aplicados **por torneo**: solo el `admin` de ese torneo accede a su panel y endpoints de administración.

---

## 12. Seguridad — OWASP Top 10 (2021) con controles concretos

> **Nivel de datos:** se manejan **datos personales** (nombre, correo, **teléfono verificado**) y, además, **datos de integridad competitiva** (pronósticos, puntos, ranking) con potencial valor económico si hay premios. Tratar teléfono/correo como **PII** y los pronósticos/puntos como datos de **integridad crítica**.

### Principio rector de arquitectura
Con **GitHub Pages + Google Sheets**, el frontend es **público y no confiable**: cualquiera puede leer su código, sus variables `VITE_*` y llamar directamente al endpoint. La **única frontera de seguridad real es Apps Script (o el proxy serverless)**. Regla absoluta: **toda autorización, validación, control de fecha límite y cálculo de puntos ocurre del lado del servidor; el cliente nunca es fuente de verdad.**

### Riesgo específico de quiniela (integridad del juego) — máxima prioridad
- **Anti-trampa de fecha límite (15 min antes del inicio):** `enviarPronostico` debe **rechazar server-side** cualquier pronóstico recibido después de la `fecha_limite` del partido, definida como **`inicio_partido` − 15 minutos** y evaluada con la **hora del servidor** (nunca la del cliente). La `fecha_limite` se **deriva en el servidor** del `inicio_partido` oficial del proveedor de API, de modo que el cliente no pueda alterarla. Un jugador no puede pronosticar un partido cuyo cierre (15 min antes del inicio) ya ocurrió, ni uno ya iniciado/terminado.
- **Inmutabilidad tras cierre:** una vez bloqueado (`bloqueado=true`) o calificado, el pronóstico **no puede modificarse**, ni siquiera por el mismo usuario.
- **No filtrar pronósticos ajenos antes del cierre:** los endpoints de lectura **no** deben exponer los pronósticos de otros participantes mientras la jornada esté abierta (evita copiar). Solo se revelan tras la fecha límite.
- **Resultados y cálculo de puntos solo server-side:** los resultados oficiales se reciben únicamente desde el proveedor de API verificado (sección 5) y `calificarJornada` es una operación interna del servidor; los puntos nunca se aceptan desde el cliente.
- **Idempotencia y auditoría:** calificar una jornada debe ser idempotente y quedar registrado en `RegistroAuditoria` (quién/qué proceso, cuándo, qué cambió).

### Riesgo de multi-torneo y códigos de invitación
- **Privacidad de torneos (solo por invitación):** todos los torneos son **privados**; el backend **no** expone ningún endpoint de listado o descubrimiento de torneos ajenos, y `obtenerMisQuinielas` devuelve únicamente torneos donde el usuario ya tiene `Participacion`. La **única** vía de ingreso es `unirseQuiniela(codigo)` con un código válido recibido por invitación.
- **Generación del código server-side:** el `codigo` de 6 caracteres alfanuméricos se genera **solo en el servidor** con aleatoriedad adecuada, verificando **colisión** contra `Quinielas.codigo` y reintentando hasta obtener uno único. El cliente nunca propone ni fija el código. (Opcional: excluir caracteres ambiguos como `0/O`, `1/I/l` para evitar errores de tecleo.)
- **Anti-enumeración/fuerza bruta de códigos:** `unirseQuiniela` debe tener **rate limiting estricto** por usuario/IP (con `CacheService`/`PropertiesService`) y backoff ante intentos fallidos repetidos, para impedir adivinar códigos por fuerza bruta. Registrar ráfagas de intentos fallidos en `RegistroAuditoria` y alertar.
- **Validación de formato:** rechazar cualquier `codigo` que no cumpla `^[A-Za-z0-9]{6}$` antes de tocar datos (lista blanca de caracteres).
- **Autorización por pertenencia a torneo:** toda lectura/escritura se restringe a torneos donde el usuario tiene una `Participacion`. Un usuario **no** puede leer jornadas, pronósticos, ranking ni avisos de torneos a los que no pertenece. El rol admin se evalúa **por torneo** (`Participaciones.rol_en_quiniela`), nunca global.
- **Aislamiento entre torneos:** el `id_quiniela` objetivo se valida server-side contra la pertenencia del usuario; no confiar en el `id_quiniela` enviado por el cliente sin verificar membresía.

### A01 — Broken Access Control
- El cliente **nunca** envía `id_usuario` confiable. El servidor deriva la identidad **del `id_token` verificado**; el **rol operativo** se deriva **por torneo** desde `Participaciones` (no se confía en un rol enviado por el cliente).
- Autorización por propiedad de recurso: un jugador solo lee/escribe **sus** `Pronosticos` y su `Usuarios`, y **solo dentro de torneos donde participa**. Operaciones de admin (crear jornadas, publicar avisos, gestionar participantes) exigen `rol_en_quiniela=admin` **en ese torneo**, verificado server-side.
- **Denegar por defecto** (HTTP 403). No exponer endpoints de admin en la ruta pública sin verificación de rol.
- No confiar en ocultar el panel admin en el UI como medida de acceso (es solo cosmético).

### A02 — Cryptographic Failures (datos sensibles)
- **HTTPS obligatorio** extremo a extremo; nunca contenido mixto `http://`.
- **PII en reposo:** teléfono y correo normalizados y minimizados; considerar **hash** (SHA-256 con sal) del teléfono para búsquedas si el negocio no requiere el número en claro.
- **Tokens:** no persistir el `id_token` en `localStorage` (riesgo XSS). Mantener en memoria; renovar vía Google Identity. Sesión propia con cookie `HttpOnly`+`Secure`+`SameSite=Strict` si el proxy lo permite.
- No registrar PII ni tokens en logs ni en mensajes de error.

### A03 — Injection (incluye Formula/CSV Injection en Sheets)
- **CSV/Formula injection (crítico aquí):** todo valor de usuario escrito en celda que empiece con `=`, `+`, `-`, `@`, tab o CR debe **sanitizarse** (anteponer `'`) o rechazarse. Aplica a `nombre_visible`, **nombre del torneo**, correo y cualquier texto libre (avisos, notas).
- **Validación estricta server-side:** pronósticos numéricos no negativos y dentro de rango razonable; `pronostico_resultado` ∈ {`1`,`X`,`2`}; `id_partido`/`id_jornada` deben existir y pertenecer a la jornada abierta (lista blanca); **`codigo` de torneo ∈ `^[A-Za-z0-9]{6}$` y debe existir**; correo válido; teléfono E.164.
- Renderizar datos de usuario con `textContent`, nunca `innerHTML`. **i18n:** interpolar variables en las cadenas traducidas de forma segura (sin `innerHTML`); las traducciones no deben permitir inyección de marcado.

### A04 — Insecure Design
- **Rate limiting / anti-abuso** en el endpoint (por usuario/IP con `CacheService`/`PropertiesService`), y **especialmente** en envío de OTP/SMS (anti-bombing y control de costos) y en **`unirseQuiniela`** (adivinación de códigos por fuerza bruta).
- **Modelo de amenazas documentado:** anónimo en internet, jugador malicioso (pronosticar tarde, editar tras cierre, ver pronósticos ajenos, autoasignarse puntos/admin, **adivinar códigos de torneo o acceder a torneos donde no participa**), abuso de SMS, scraping del ranking, manipulación de la sincronización con el proveedor de API de resultados.
- OTP de un solo uso, expirable (5–10 min), con límite de intentos y backoff.
- Lógica de fecha límite, **generación de códigos** y cálculo de puntos **centralizada** en el servidor, no replicada/confiada en el cliente.

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
- [ ] **Fecha límite server-side (15 min antes del inicio):** se rechaza todo pronóstico posterior a `fecha_limite = inicio_partido − 15 min` usando la hora del servidor; la `fecha_limite` se **deriva server-side** del `inicio_partido` oficial.
- [ ] Pronósticos inmutables tras cierre/calificación; sin edición posterior.
- [ ] Pronósticos ajenos **no** visibles antes de la fecha límite.
- [ ] Resultados oficiales sincronizados **solo** desde el proveedor de API verificado; cálculo de puntos **solo** server-side; nunca aceptados del cliente.
- [ ] Identidad derivada del **`id_token` verificado** (firma, `aud`, `iss`, `exp`); el **rol se evalúa por torneo** desde `Participaciones`; el cliente nunca envía `id_usuario`/`rol` confiables.
- [ ] Autorización por propiedad de recurso **y por pertenencia a torneo**; denegar por defecto; endpoints admin protegidos por `rol_en_quiniela=admin` en ese torneo.
- [ ] **Códigos de torneo:** generados server-side, únicos (sin colisión), validados con `^[A-Za-z0-9]{6}$`; `unirseQuiniela` con rate limiting/backoff anti-fuerza bruta. **Torneos privados:** sin endpoint de listado/descubrimiento; el único ingreso es por invitación con el código.
- [ ] **Sanitización anti-formula injection** en toda celda con input de usuario (`= + - @`).
- [ ] Validación server-side: pronóstico en rango, `resultado` ∈ {1,X,2}, ids en lista blanca de la jornada abierta, correo, teléfono E.164.
- [ ] CORS al origen de Pages; CSP en `index.html`; orígenes del ID de Cliente limitados.
- [ ] Rate limiting en endpoint y en OTP/SMS; OTP de un solo uso/expirable; `telefono_verificado` y `rol` solo server-side.
- [ ] Tokens fuera de `localStorage`; HTTPS estricto; sin contenido mixto.
- [ ] Publicidad en `<iframe sandbox>`; `npm ci` + `npm audit` en CI; Dependabot activo.
- [ ] `RegistroAuditoria` de eventos de integridad y seguridad sin PII; borrado de cuenta disponible.
- [ ] Sincronización con el proveedor de API de resultados autenticada/validada; clave de API solo en el servidor.

---

## 13. Protección de propiedad intelectual (PI) y disuasión legal

> **Objetivo:** establecer una **capa de disuasión legal** y de **señalización explícita de intención de protección** sobre el código y el contenido del proyecto. **Advertencia de alcance:** un frontend estático servido como un único `index.html` es **inherentemente público** (su código viaja al navegador y puede leerse); estas medidas **no garantizan protección técnica absoluta**, pero sí afirman la titularidad y restringen usos no autorizados en marcos como la **Directiva Europea de Derechos de Autor en el Mercado Único Digital (DSM, 2019/790)** —en particular su régimen de *opt-out* de minería de textos y datos (art. 4)— y estándares emergentes de la industria (W3C TDM Reservation Protocol, convenciones `noai`/`noindex`).

### (a) Aviso de copyright y licencia restrictiva
- **Licencia propietaria — Todos los derechos reservados.** El repositorio incluye un archivo **`LICENSE`** con una licencia restrictiva que **prohíbe la copia, redistribución, ingeniería inversa con fines de reutilización y la creación de obras derivadas** sin autorización **expresa y por escrito** del titular. No es software libre ni de código abierto.
- **Aviso de copyright explícito** en el proyecto (al menos en `LICENSE`, en el `README` si existe, y como **comentario en la cabecera del `index.html`** —que sobrevive aunque se minifique con `format: { comments: false }` si se marca como comentario legal `/*! ... */`, ver más abajo):

```text
Copyright (c) <AÑO> <TITULAR DE DERECHOS>. Todos los derechos reservados.

Este software y su contenido (código, diseño, tokens visuales, textos y datos)
son propiedad de <TITULAR DE DERECHOS> y están protegidos por las leyes de
derechos de autor aplicables. Queda PROHIBIDA su copia, distribución,
modificación, descompilación, ingeniería inversa orientada a reutilización,
o la creación de obras derivadas, total o parcialmente, por cualquier medio,
sin autorización previa y por escrito del titular. El acceso al código por ser
un frontend público NO concede licencia ni derecho de uso alguno.

Identificador SPDX: LicenseRef-Proprietary-AllRightsReserved
```

- **Cabecera legal en el `index.html`** (comentario *banner* preservable por el minificador como comentario legal):

```html
<!--! © <AÑO> <TITULAR DE DERECHOS>. Todos los derechos reservados.
      Licencia propietaria. Prohibida la copia, redistribución o uso derivado
      sin autorización por escrito. SPDX: LicenseRef-Proprietary-AllRightsReserved -->
```

### (b) `robots.txt` con bloqueo de crawlers y directivas `noai` / `noindex`
- Crear **`public/robots.txt`** (se publica en la **raíz** del sitio) con **`Disallow`** para crawlers no autorizados —incluyendo de forma **explícita** los rastreadores de IA conocidos— y las directivas **`noai`** y **`noindex`** como señalización explícita:

```text
# robots.txt — © <AÑO> <TITULAR DE DERECHOS>. Todos los derechos reservados.
# Señalización de NO consentimiento para indexación ni minería/entrenamiento de IA.

# --- Rastreadores de IA / entrenamiento de modelos: acceso denegado ---
User-agent: GPTBot
User-agent: OAI-SearchBot
User-agent: ChatGPT-User
User-agent: Google-Extended
User-agent: ClaudeBot
User-agent: anthropic-ai
User-agent: CCBot
User-agent: PerplexityBot
User-agent: Bytespider
User-agent: Amazonbot
User-agent: Applebot-Extended
User-agent: Meta-ExternalAgent
User-agent: FacebookBot
User-agent: Diffbot
User-agent: cohere-ai
Disallow: /

# --- Directivas explícitas de señalización (opt-out de IA e indexación) ---
noai: 1
noindex: 1
noimageai: 1

# --- Resto de crawlers ---
User-agent: *
Disallow: /
noai: 1
noindex: 1
```

> Nota técnica: `noai`/`noindex` **dentro de `robots.txt` no son estándar universal** y no todos los rastreadores los honran; se incluyen como **señalización explícita de intención** (exigida en el criterio). El **`noindex` con efecto real** se logra con la *meta robots* y la cabecera `X-Robots-Tag` (abajo). El cumplimiento del *opt-out* depende del operador del crawler.

### (c) Meta-tags en el `<head>`: opt-out de TDM y señalización anti-IA/indexación
Incluir en el `<head>` del `index.html` (junto a la CSP de §12 A05):

```html
<!-- Opt-out de Text & Data Mining (W3C TDMRep) -->
<meta name="tdm-reservation" content="1">
<!-- Opcional: política de TDM legible por máquina -->
<meta name="tdm-policy" content="https://<dominio>/tdm-policy.json">

<!-- Señalización anti-IA e indexación -->
<meta name="robots" content="noai, noimageai, noindex, nofollow, noarchive">
<meta name="googlebot" content="noindex, nofollow">

<!-- Titularidad -->
<meta name="copyright" content="© <AÑO> <TITULAR DE DERECHOS>. Todos los derechos reservados.">
<meta name="author" content="<TITULAR DE DERECHOS>">
```

- **Refuerzo por cabecera HTTP (recomendado donde el hosting/proxy lo permita):** emitir `X-Robots-Tag: noai, noindex, nofollow` y, para TDM, el encabezado `tdm-reservation: 1` (GitHub Pages no permite cabeceras personalizadas; si se requiere, anteponer un proxy/CDN o servir vía el proxy serverless).

### (d) Build de producción endurecido (menor legibilidad del bundle)
Configurado en `vite.config.js` (sección 10), como parte de la PI:
- **Sin source maps** en producción: `build.sourcemap = false` (no se publican archivos `.map` que reconstruyan el código fuente).
- **Supresión de comentarios** y **nombres no descriptivos**: `terserOptions.format.comments = false` y `mangle = true` (salvo el comentario legal `/*! ... */` / `<!--! ... -->`, que se conserva).
- **Inline total** en un único `index.html` (`vite-plugin-singlefile`), lo que además elimina rutas de assets y dificulta el mapeo del proyecto.
- `drop_console`/`drop_debugger` activados para no filtrar trazas internas.

> Estas medidas **reducen la legibilidad del código compilado sin comprometer la funcionalidad**; no son cifrado ni ofuscación fuerte, y un actor determinado puede revertirlas. Su valor es **disuasorio y probatorio** (demostrar intención de protección), no de impedimento técnico absoluto.

### Checklist de PI (criterio de aceptación)
- [ ] `LICENSE` propietario (**Todos los derechos reservados**) presente; prohíbe copia/redistribución/derivados sin autorización.
- [ ] Aviso de copyright en `LICENSE` y como **comentario legal** en el `index.html` (preservado pese a la minificación).
- [ ] `public/robots.txt` en la **raíz** con `Disallow: /` para crawlers de IA conocidos y `User-agent: *`, más directivas **`noai`** y **`noindex`** explícitas.
- [ ] Meta-tags en `<head>`: **`tdm-reservation: 1`**, `robots` con `noai, noindex, nofollow` y meta de copyright/autor.
- [ ] (Si el hosting lo permite) cabecera `X-Robots-Tag: noai, noindex` y `tdm-reservation: 1`.
- [ ] Build sin **source maps**; comentarios suprimidos y `mangle` activo; salida en un único `index.html` (sin `assets/` ni `.map`).
- [ ] Documentado el **alcance/límite**: el frontend es público; estas medidas son **disuasión legal + señalización**, no protección técnica absoluta.

---

## 14. Orden de implementación sugerido

1. Scaffolding **en un único `index.html`** (Vite + `vite-plugin-singlefile`): `<style>` embebido con **tokens + temas oscuro/claro**, **marco móvil con radio de aspecto**, **i18n es/en como objetos JS embebidos**, `ProveedorTema`/`ProveedorIdioma` y un **enrutador en memoria**; layout con encabezado/BarraJornadas/nav inferior fijos. Incluir desde el inicio `<SelectorTema>` y `<SelectorIdioma>` con persistencia. **Configurar la PI (sección 13) desde el primer commit:** `LICENSE` propietario, `public/robots.txt` (`Disallow` + `noai`/`noindex`), meta-tags de `<head>` (`tdm-reservation: 1`, `robots noai,noindex`, copyright) y `vite.config.js` endurecido (sin sourcemaps, sin comentarios, `mangle`, salida de un solo archivo).
2. Pantalla **Jugar** con **datos de prueba** (jornada, partidos, entrada de pronóstico) hasta lograr paridad visual y funcional (sección 11), validando ambos temas y ambos idiomas.
3. Apps Script + estructura de Sheets (incluida `Participaciones`); lectura (`obtenerMisQuinielas`, `obtenerJornadaActiva`, `obtenerPartidosJornada`, `obtenerMisPronosticos`). **Aplicar desde aquí los controles de la sección 12** (validación, CORS, sanitización, **fecha límite server-side**).
4. Integración con el proveedor de API de resultados deportivos en vivo: sincronización periódica/webhook que actualiza `Partidos` (estado, marcador, horarios) — ver sección 5.
5. Auth (Google → alternativa correo) + verificación de teléfono por OTP + **roles por torneo**, con verificación de `id_token` server-side.
6. **Crear y unirse a torneos:** `crearQuiniela` (generación del **código único de 6 caracteres** server-side) y `unirseQuiniela(codigo)` con validación/rate limiting; selector de torneo activo y cambio de contexto.
7. Envío de pronósticos con bloqueo por fecha límite (**15 min antes del inicio de cada partido**); lectura del ranking (`obtenerTablaPosiciones`) y del **estado de jugadores** (`obtenerEstadoJugadores`, §4.4.1: Atrasado / En Riesgo / Al día) del torneo activo.
8. Calificación automática de jornadas: `calificarJornada` se dispara cuando el proveedor de API marca los partidos como `finalizado` (cálculo de puntos) + `RegistroAuditoria`. Panel **admin** (por torneo) para gestión de jornadas/participantes/avisos.
9. EspacioPublicidad en iframe sandbox; CI/CD a GitHub Pages (`npm ci` + `npm audit`); estados de carga/vacío/cerrado/error; pulido del marco responsive (móvil + radio de aspecto en desktop).

---

### Definición de "hecho"
La web app **administra la quiniela**: el usuario autenticado (Google + teléfono verificado, o correo) puede **crear múltiples torneos privados y unirse a otros, exclusivamente por invitación, mediante un código único de 6 caracteres alfanuméricos**, y en el torneo activo captura pronósticos de la jornada abierta que se **bloquean automáticamente 15 minutos antes del inicio de cada partido** (`fecha_limite = inicio_partido − 15 min`, validado server-side); los resultados oficiales se **sincronizan automáticamente desde un proveedor de API de resultados deportivos en vivo** y disparan el **cálculo de puntos** según el reglamento; el **ranking** de participantes se actualiza y se muestra —junto con el **estado de cada jugador** (**Atrasado**, **En Riesgo** o **Al día**, §4.4.1)—, todo respetando los **roles por torneo**. La interfaz es **exclusivamente móvil** (en desktop conserva el **radio de aspecto** en un marco centrado), soporta **tema oscuro/claro** (oscuro por defecto) e **idiomas español/inglés** (español por defecto), ambos conmutables y persistentes, conserva la estética de la captura (sección 11) y pasa el **checklist de seguridad OWASP** (sección 12), desplegada en GitHub Pages bajo HTTPS con CORS y CSP correctos, como un **único `index.html` autocontenido (SPA estricta de archivo único)** —con toda navegación resuelta **en memoria**, sin rutas ni archivos HTML adicionales— y con la **capa de protección de PI activa** (licencia restrictiva *Todos los derechos reservados*, `robots.txt` con `noai`/`noindex`, meta `tdm-reservation`, build sin source maps; sección 13).
