# CLAUDE.md — Quiniela Web App (Liga de Fútbol)

> Archivo principal de contexto para **Claude Code**.
> **Misión del producto:** construir una **web app** cuyo objetivo es **administrar el juego de quiniela de una liga de fútbol**: los usuarios **crean múltiples torneos privados o se unen a otros por invitación mediante un código único de 6 caracteres**, pronostican los resultados de las jornadas, el sistema **cierra la captura de pronósticos de cada partido 15 minutos antes de su inicio**, calcula puntos contra los resultados oficiales y mantiene una **tabla de posiciones** de participantes por torneo.
> **Arquitectura:** frontend estático en **GitHub Pages** servido como **aplicación de una sola página en un único archivo HTML (SPA estricta de archivo único)** + **Google Sheets como base de datos**. Autenticación vía **Google Auth** (principal, con número de celular verificado) y **correo electrónico** como alternativa.
> **Restricción de archivo único (NO negociable):** **todo el cliente reside en un solo `index.html`**; todas las vistas, paneles, componentes y flujos (autenticación, torneos, jornadas, pronósticos, ranking, estado de jugadores, configuración) se renderizan **dinámicamente dentro de ese mismo archivo**, mediante manipulación del DOM y **enrutamiento en memoria**, sin rutas separadas ni archivos HTML adicionales (ver sección 0 y §15). El backend en **Google Apps Script** es una pieza de servidor independiente y no forma parte de ese HTML.
> **Base visual (UI/UX):** se replica el estilo de la captura de referencia (interfaz deportiva móvil con su **propia identidad visual**) — encabezado con menú desplegable, navegación por fechas/jornadas, espacio de publicidad y barra inferior — **pero adaptado al dominio de quiniela** (pronósticos, ranking, jornadas), no a un simple marcador. La interfaz es **exclusivamente móvil** (en desktop conserva el radio de aspecto en un marco centrado), con **tema oscuro/claro** (oscuro por defecto) e **idiomas español/inglés** (español por defecto).

---

## 0. Reglas para Claude Code (leer primero)

- **La misión manda:** todo lo que se construya sirve a **administrar la quiniela**. Nunca implementar funcionalidad de "marcador" que no sirva al juego de pronósticos.
- **Arquitectura de un solo archivo HTML (SPA estricta) — restricción NO negociable.** **Todo el cliente del proyecto debe residir en un único archivo `index.html`.** Todos los componentes, vistas, paneles y flujos (autenticación, torneos, jornadas, pronósticos, ranking, estado de jugadores, configuración) se renderizan **dinámicamente dentro de ese mismo archivo**, respondiendo a la interacción del usuario, **sin generar rutas separadas ni archivos HTML adicionales**. El `index.html` es el **contenedor absoluto** del cliente: el marcado, los estilos (tokens de la sección 3 en un `<style>` embebido), la lógica de la app y los diccionarios de idioma (sección 9.1, como objetos JS embebidos) viven dentro de él. Toda navegación, transición entre vistas o carga de contenido se resuelve **en el cliente con JavaScript** (manipulación del DOM, componentes condicionales o **enrutamiento en memoria**), **nunca** cargando páginas externas. Esta restricción aplica **tanto al entorno de desarrollo como al artefacto final** desplegado en GitHub Pages. **Excepción:** el backend **Google Apps Script** (`Codigo.gs`) es una pieza de servidor que se despliega aparte en Google y **no** forma parte del HTML único; y los archivos **no-HTML** estrictamente necesarios por otras secciones (p. ej. `robots.txt`, `LICENSE`, configuración de build) pueden existir como archivos sueltos, ya que no son vistas de la aplicación.
- **Exclusivamente móvil (mobile-only).** La referencia es una vista de teléfono (~390–430px de ancho lógico). **El diseño es solo para móvil:** no se construyen vistas alternativas de tablet/desktop. Cuando la app se accede desde **desktop o pantallas grandes, se mantiene el radio de aspecto** del teléfono renderizando un **marco centrado** (ancho máx. `--ancho-app-max`, radio de aspecto `--aspecto-app`) sobre un fondo neutro (letterbox). Nunca se ensancha el layout a varias columnas ni se reflowea a escritorio.
- **Tema oscuro y claro.** La app soporta **dos temas: oscuro (por defecto) y claro**, conmutables por el usuario y persistentes entre sesiones. Todo color proviene de tokens (sección 3); el cambio de tema solo altera tokens de color, nunca la estructura.
- **Bilingüe (es/en).** La UI/UX soporta **exactamente dos idiomas: español (por defecto) e inglés** como alternativa, conmutables por el usuario y persistentes. Todo texto visible se obtiene de archivos de recursos de idioma (sección 9.1); no se escriben cadenas de texto "a mano" en los componentes.
- **Multi-torneo privado + unirse por invitación.** La app permite **crear múltiples juegos/torneos**, y todos son de carácter **privado**: **no existen torneos públicos ni son descubribles o listables**. El único modo de ingresar a un torneo es **por invitación**, compartiendo el **código único de 6 caracteres alfanuméricos** (solo letras y números) asignado al torneo. Un usuario participa en varios torneos a la vez (sección 6).
- **Créditos y premios centralizados + terminología (fuente única §14).** **Todo** lo relativo a **créditos, costos, bolsa, pagos y premios** vive **exclusivamente en la §14**; ninguna otra sección define reglas económicas (solo la **referencian**). Al crear un torneo se elige su **tipo**: **Competitivo** (con créditos, costo de inscripción y premios) o **Recreativo** (gratuito, sin recompensas). En **toda** la UI, el valor/saldo/costo/premio se denomina **únicamente «Créditos»**: queda **prohibido** usar en el texto visible palabras como «dinero», «pesos», «dólares» o «efectivo», ni símbolos de moneda ($, €), para denominar valor dentro de la app (prevención de contingencias legales). Detalle completo y cerrado en **§14**.
- **Frontend 100% estático y de archivo único.** GitHub Pages no ejecuta backend y **sirve un solo `index.html` autocontenido**. Toda lógica de escritura/validación/cálculo de puntos va a **Google Apps Script** desplegado como Web App (endpoint REST) o a un proxy serverless.
- **Nunca** expongas credenciales de cuenta de servicio, secretos de OAuth o claves de API con permisos de escritura en el cliente. El cliente solo usa el **ID de Cliente** público de Google Identity Services.
- **Paridad visual es criterio de aceptación** (estética de la captura), pero **subordinada a la misión**. Usa los tokens de diseño de la sección 3. Valida al final contra las listas de verificación (secciones 11 y 12).
- Código limpio, comentado en español, sin dependencias innecesarias. **JavaScript puro (Vanilla JS) embebido en el `index.html` es la vía preferida** para honrar la restricción de archivo único en desarrollo y producción. Si se usa **React**, debe autorizarse vía CDN embebido o **empaquetarse con Vite + `vite-plugin-singlefile`** de modo que el **build de producción produzca exactamente un `index.html` autocontenido** (sin chunks ni `assets/` externos). No usar nada que requiera servidor en runtime ni que rompa el principio de un solo archivo.
- **Todo el código, nombres de variables, funciones, componentes, pestañas y columnas de datos deben estar exclusivamente en español** (ver convención de nombres en cada sección). **Excepción:** el **texto visible de la UI** se externaliza en los archivos de idioma `es`/`en` (sección 9.1); los identificadores del código siguen siendo en español, pero las cadenas que ve el usuario viven en los recursos de traducción.

---

## 1. Resumen del producto

Web app para **administrar y jugar una quiniela** de una liga de fútbol. Funciones núcleo:

- **Crear y unirse a torneos (privados):** un usuario puede **crear múltiples juegos/torneos** (cada uno con su nombre, temporada y **formato/reglamento de puntuación**, §5). **Todos los torneos son privados:** solo se puede ingresar **por invitación**, compartiendo el **código único de 6 caracteres alfanuméricos** asignado; no hay directorio público ni búsqueda de torneos ajenos. Para **unirse a un torneo de otra persona** se introduce dicho código. La vista activa siempre corresponde a uno de los torneos en los que participa.
- **Jugar pronósticos:** el participante ve los partidos de la **jornada** abierta y captura su pronóstico **capturando el marcador del partido** (goles local–visitante, del que se derivan el signo 1-X-2 y el acierto exacto, §5.1), lo cual solo es posible **hasta 15 minutos antes del inicio de cada partido** (ese instante es la `fecha_limite` del partido).
- **Cierre por fecha límite:** la captura de cada partido se cierra **15 minutos antes de su hora de inicio** (`fecha_limite = inicio_partido − 15 min`); al llegar ese momento, el pronóstico de ese partido se bloquea y ya no puede editarse. El cierre es **por partido**, de forma independiente, no por jornada completa.
- **Resultados oficiales:** se capturan los marcadores reales de cada partido (ver sección 6 — fuente de datos vía API externo).
- **Cálculo de puntos:** el sistema asigna puntos **analizando el marcador pronosticado de cada jugador frente al resultado oficial**, según el **formato del torneo** (**Casual**, **Incremental** o **Personalizado**) definido de forma **única y cerrada en la sección 5**, y actualiza el marcador de cada participante.
- **Tabla de posiciones (ranking):** clasificación de participantes por puntos acumulados, por jornada y general, **por torneo**.
- **Estado de los jugadores (por torneo):** cada participante recibe un **estado de cumplimiento de pronósticos** en tres categorías —**Atrasado**, **En Riesgo** o **Al día**—, calculado server-side a partir de sus partidos sin pronosticar y las fechas límite de cada partido (definición en §4.4.1 y §6).
- **Roles (resumen; definición canónica y única en §8):** el **rol base de todo usuario es Jugador**. Cada torneo puede tener además un **equipo administrativo** con cuatro roles —**Comisionado**, **Tesorero**, **Operador** y **Vocal**—, asignados **por torneo** (nunca de forma global). El **creador** es, por defecto, el **Comisionado** (único por torneo y único facultado para asignar/transferir roles administrativos y expulsar jugadores). Los administradores pueden optar por **jugar además de gestionar** o por **solo gestionar** (sin jugar). La definición completa de roles, permisos, cardinalidad y reglas de control vive **exclusivamente en la §8**; el resto de secciones se subordina a ella.
- **Créditos y premios (solo torneos Competitivos):** un torneo puede ser **Competitivo** (gestiona **Créditos**: costo de inscripción, **Bolsa Única** y distribución de premios) o **Recreativo** (gratuito, sin recompensas). **Toda** la mecánica económica —terminología, pagos al **Tesorero**, **Bolsa**, **calculadora de distribución** y el **Premio al Campeón**— se define de forma **única y centralizada en §14**. Esta sección solo la menciona; no la redefine.

---

## 2. Stack técnico

| Capa | Tecnología |
|---|---|
| Frontend | **Un único `index.html` (SPA estricta de archivo único).** Vanilla JS embebido (preferido) **o** React vía CDN / `vite-plugin-singlefile`; todo el marcado, estilos y lógica viven en el mismo archivo |
| Estilos | Variables CSS (tokens de diseño) en un **`<style>` embebido** en el `index.html` + **temas oscuro/claro** vía `data-tema` (Tailwind solo vía CDN si se desea; sin CSS modules) |
| Internacionalización | Diccionarios `es`/`en` como **objetos JS embebidos** en el `index.html` + utilidad propia ligera; idioma persistido |
| Marco móvil | Layout de ancho fijo (`--ancho-app-max`) con radio de aspecto en desktop (sin reflow a escritorio) |
| Hosting | GitHub Pages sirviendo **un único `index.html` autocontenido** (rama `gh-pages` o `/docs`) |
| Datos (lectura) | **Google Apps Script GET** (única vía de lectura): valida token, **pertenencia a torneo** y **fecha límite**. **No** se lee Google Sheets directamente desde el cliente (ver §10 y §13) |
| Datos (escritura + cálculo de puntos) | Google Apps Script Web App (POST con validación de token y de fecha límite) |
| Datos de partidos (en vivo, pasados y futuros) | Proveedor externo de API de resultados deportivos en vivo (ver sección 6) |
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

> **Tema por defecto:** oscuro. La preferencia del usuario se aplica fijando `data-tema` en `<html>` y se persiste (ver sección 9.2). Respetar `prefers-color-scheme` solo como valor inicial si el usuario aún no eligió; una vez elegido, manda su preferencia.

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
  - **Perfil/usuario con engrane** → estado de sesión. Si logueado: avatar + acceso a perfil; si no: abre `<ModalAutenticacion>`. Aquí, los integrantes del **equipo administrativo** del torneo activo (Comisionado, Tesorero, Operador o Vocal, §8) ven su acceso al **panel de administración**, que expone **únicamente** las funciones permitidas por su rol (**principio de menor privilegio**, §8 y §13). El **panel del Tesorero** (pagos/inscripciones y **Bolsa Única**) y la **calculadora de distribución de premios** forman parte de este panel y se especifican de forma única en **§14**. Desde aquí se accede a **preferencias**: `<SelectorTema>` (oscuro/claro) y `<SelectorIdioma>` (es/en).
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
- Resumen del jugador: **posición en el ranking**, **puntos** (jornada y total), **pronósticos pendientes** de la jornada abierta y, si aplica, **Créditos/premios** (solo torneos Competitivos, §14) o racha.
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
- La insignia solo expone la **categoría** de estado; **nunca** el contenido de los pronósticos ajenos (respeta §13: no filtrar pronósticos antes del cierre).

**Cálculo:** se deriva **server-side** de `Pronosticos` + `Partidos.fecha_limite` contra la **hora del servidor**. Como las categorías dependen de una ventana de **24 h relativa al momento de consulta**, el estado se calcula al leer (o se cachea con TTL corto); no se congela en `Posiciones`.

### 4.5 `<PartidosAPronosticar>` (partidos de la jornada)
- Encabezado uppercase gris (ej. "PARTIDOS — JORNADA 12").
- Lista de partidos de la jornada. Patrón de fila repetible (`<FilaPartido>`):
  - Escudo + **equipo local** vs **equipo visitante** + escudo.
  - **Hora límite** (15 minutos antes del inicio) e **inicio del partido**, con estado (abierto / cerrado / en juego / final); mostrar cuenta regresiva al cierre cuando falte poco.
  - **Entrada de pronóstico** (inputs de marcador y/o 1-X-2) cuando está **abierto**; **bloqueado** (solo lectura) cuando pasó la fecha límite.
  - Si ya hay resultado oficial: mostrar marcador real, el pronóstico del jugador y **puntos obtenidos** (valor final ya multiplicado según formato y fase, §5; resaltar en verde si acertó). En formato **Incremental**, los partidos de eliminación directa indican su **multiplicador de ronda** (×2…×5).
- Separador inferior sutil entre filas.

### 4.6 `<EspacioPublicidad>`
- Etiqueta pequeña "Publicidad".
- Banner ~ancho completo, alto `--alto-publicidad`, anclado sobre la nav inferior. **Espacio configurable** (imagen/iframe). No tapa contenido ni la nav.

### 4.7 `<NavInferior>` (fija) — 5 ítems adaptados a quiniela
1. **Jugar** — captura de pronósticos de la jornada abierta (activo en rojo por defecto).
2. **En Vivo** — partidos en curso y cómo van los pronósticos.
3. **Ranking** — tabla de posiciones de participantes (con insignia de cambios/posición).
4. **Avisos** — comunicados del organizador / noticias de la liga.
5. **Reglas** — reglamento de la quiniela: **formato de juego** (Casual / Incremental / Personalizado) y **puntos por resultado correcto y marcador exacto, por fase** (§5), además de fechas límite. Los **premios y Créditos** (torneos Competitivos) se rigen **exclusivamente por la §14**.
- Ítem activo en rojo; inactivos en gris claro. Barra fija, no hace scroll.

### 4.8 Componentes superpuestos (modales y selectores)
- **`<ModalCrearQuiniela>`** — formulario para crear torneo (nombre, temporada y **selección de formato**: Casual / Incremental / Personalizado, §5). Si se elige **Personalizado**, el formulario exige capturar los **puntos por resultado y por marcador exacto de cada fase ANTES de lanzar** (§5.3.3 y §5.5). Al confirmar, el **servidor genera el código único de 6 caracteres**, **congela las reglas de puntuación** y el modal muestra el código con opción de copiar/compartir. En este mismo modal se elige el **tipo de torneo** (**Competitivo**/**Recreativo**); si es **Competitivo**, se captura el **costo de inscripción en Créditos** (mecánica y validación en **§14**).
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

## 5. Mecánica de juego: tipos de torneo y puntuación

> **Fuente única y cerrada de la puntuación.** Esta sección es la **única** definición de la mecánica de puntos del producto. Ninguna otra sección redefine fórmulas: el *Resumen* (§1), la *UI/UX* (§4), el *Modelo de datos* (§6), la *Capa de acceso a datos* (§7) y la *Seguridad* (§13) **se subordinan y remiten a este algoritmo**. Toda la mecánica se calcula **siempre analizando el marcador pronosticado por cada jugador en cada partido** y comparándolo, **server-side**, contra el resultado oficial de `Partidos`.
>
> **Nota (puntos ≠ Créditos):** esta sección gobierna **puntos deportivos** (ranking). Los **Créditos y premios** son un eje **independiente** definido **exclusivamente en §14**: los puntos determinan **posiciones**, y la §14 decide cómo (y si) esas posiciones se traducen en premios (incluido el **Premio al Campeón**, que **no** premia al de más puntos).

### 5.1 Conceptos base (comunes a todos los formatos)

Para cada `Pronostico` de un `Partido` con resultado oficial (`goles_local`, `goles_visitante`):

- **Acierto de resultado (signo 1-X-2):** el signo pronosticado coincide con el signo oficial —gana local (`1`), empate (`X`) o gana visitante (`2`)—. El "resultado correcto" abarca tanto **ganador** como **empate**.
- **Acierto de marcador exacto:** `pronostico_local == goles_local` **y** `pronostico_visitante == goles_visitante`. El marcador exacto **siempre implica** acierto de resultado (no existe exacto con signo incorrecto).

Valores base de puntos por partido (antes de cualquier multiplicador):

- `P_RES` = puntos por **resultado correcto** (signo 1-X-2).
- `P_EXACTO` = puntos **adicionales** por **marcador exacto** (se suman a `P_RES`).

```text
base_partido = (acierto_resultado ? P_RES : 0) + (acierto_exacto ? P_EXACTO : 0)
```

Con los valores estándar `P_RES = 1` y `P_EXACTO = 3`, un **partido perfecto** otorga `1 + 3 = 4` puntos (máximo base por partido).

### 5.2 Fases del torneo

Cada `Partido` pertenece a una **fase** (`Partidos.fase`), que el sistema normaliza desde el proveedor de API (§6) o asigna el **Operador** del torneo (o el Comisionado en su defecto, §8):

| Fase | Valor (`fase`) | Tipo |
|---|---|---|
| Fase regular / grupos | `regular` | Liga / grupos |
| Octavos de Final | `octavos` | Eliminación directa |
| Cuartos de Final | `cuartos` | Eliminación directa |
| Semifinales | `semifinal` | Eliminación directa |
| Gran Final | `final` | Eliminación directa |

> Si un torneo no contempla eliminación directa, todos sus partidos son `regular`.

### 5.3 Los tres formatos de juego

El **formato** es una propiedad del torneo (`Quinielas.formato ∈ {casual, incremental, personalizado}`), elegida por el **Comisionado** al crear el torneo (configurable por el **Operador** antes del lanzamiento, §8) y **congelada al iniciar el primer partido** (ver §5.5).

#### 5.3.1 Formato **Casual** — puntuación fija y lineal
Sistema de puntos **fijo y sin variaciones** durante todo el torneo (fase regular **y** eliminación directa). Sin multiplicadores ni ventajas por fase.

- `P_RES = 1` y `P_EXACTO = 3` en **todas** las fases.
- **Multiplicador = ×1** siempre.
- **Máximo 4 puntos por partido** perfecto.
- `puntos_partido = base_partido`.

Es la opción ideal para comunidades que buscan una experiencia relajada, predecible y equitativa de principio a fin.

#### 5.3.2 Formato **Incremental** — bonificación progresiva por ronda
Mantiene la base estándar en la fase regular; en eliminación directa, los puntos del partido se **incrementan mediante un multiplicador por ronda**, premiando los aciertos en las etapas decisivas y habilitando remontadas en la tabla.

- **Fase regular / grupos:** `P_RES = 1`, `P_EXACTO = 3`, **multiplicador ×1**.
- **Eliminación directa:** misma base (`P_RES = 1`, `P_EXACTO = 3`) afectada por el **multiplicador por ronda**:

| Ronda | `fase` | Multiplicador |
|---|---|---|
| Octavos de Final | `octavos` | **×2** |
| Cuartos de Final | `cuartos` | **×3** |
| Semifinales | `semifinal` | **×4** |
| Gran Final | `final` | **×5** |

- `puntos_partido = base_partido × multiplicador(fase)`.

Ejemplos: un marcador exacto en Semifinales otorga `(1 + 3) × 4 = 16` puntos; en la Gran Final, `(1 + 3) × 5 = 20` puntos. Así, los aciertos en las rondas definitivas tienen mayor peso estratégico y el torneo se mantiene vivo hasta el último segundo.

#### 5.3.3 Formato **Personalizado** — control total del organizador
El **Operador** (o el Comisionado, §8) define **individualmente** la cantidad exacta de puntos que se otorgan por **resultado correcto** y por **marcador exacto** para **cada una de las fases** del torneo. No usa el multiplicador implícito del formato Incremental (el escalamiento se logra con los propios valores por fase).

- Por cada `fase`, el **Operador** fija `P_RES(fase)` y `P_EXACTO(fase)` (enteros ≥ 0).
- **Multiplicador = ×1** (los valores por fase ya incorporan cualquier escalamiento deseado).
- `puntos_partido = (acierto_resultado ? P_RES(fase) : 0) + (acierto_exacto ? P_EXACTO(fase) : 0)`.
- **Condición indispensable de transparencia y legalidad:** todos los valores de puntuación deben **editarse y quedar establecidos ANTES de la creación y el lanzamiento oficial del torneo**, y quedan **completamente congelados una vez que comienza el primer partido** (§5.5). No se permite modificarlos con el juego ya iniciado.

### 5.4 Algoritmo de cálculo (canónico, server-side)

`calificarJornada` (§7) aplica, por cada `Pronostico` de un `Partido` ya `finalizado`:

```text
función calcular_puntos(pronostico, partido, quiniela):
    res_ok    = signo(pronostico) == signo(partido.resultado_oficial)
    exacto_ok = (pronostico.local == partido.goles_local) y
                (pronostico.visitante == partido.goles_visitante)

    # P_RES, P_EXACTO y multiplicador según formato y fase
    # (valores EFECTIVOS ya congelados; ver §5.5)
    según quiniela.formato:
        casual:        P_RES = 1; P_EXACTO = 3; mult = 1
        incremental:   P_RES = 1; P_EXACTO = 3; mult = multiplicador_ronda(partido.fase)
        personalizado: P_RES = P_RES_fase(partido.fase)
                       P_EXACTO = P_EXACTO_fase(partido.fase)
                       mult = 1

    base   = (res_ok ? P_RES : 0) + (exacto_ok ? P_EXACTO : 0)
    puntos = base * mult
    return puntos     # se guarda en Pronosticos.puntos_otorgados (valor FINAL, ya multiplicado)

multiplicador_ronda = { regular:1, octavos:2, cuartos:3, semifinal:4, final:5 }
```

**Reglas invariantes:**
- `exacto_ok` implica `res_ok`: nunca se otorga `P_EXACTO` sin `P_RES`.
- El multiplicador aplica al **total del partido** (resultado + exacto), no solo a una de sus partes.
- `Pronosticos.puntos_otorgados` almacena el valor **final** (ya multiplicado). El ranking (`Posiciones`, §6) suma esos valores finales por jornada y total.
- Un pronóstico no enviado o bloqueado sin captura otorga **0** puntos.

### 5.5 Congelamiento de reglas (integridad y legalidad)

- Para **todos** los formatos, la configuración efectiva de puntuación (formato + `P_RES`/`P_EXACTO` por fase + multiplicadores) se **materializa y congela en el servidor al lanzar el torneo**: en el instante en que **comienza el primer partido** (`inicio_partido` mínimo del torneo), las reglas quedan **inmutables**.
- En **Personalizado**, además, los valores deben definirse **antes** de la creación/lanzamiento; el backend **rechaza** cualquier intento de edición posterior al inicio del primer partido.
- El congelamiento se refleja en `Quinielas.reglas_congeladas` y en `ReglasPuntuacion.congelado`, y queda registrado en `RegistroAuditoria` (§13). Esto garantiza una competencia transparente y equitativa, sin cambios de puntuación a mitad de torneo.

---

## 6. Modelo de datos (Google Sheets)

Una hoja (Spreadsheet) con las siguientes pestañas:

- **`Quinielas`** — `id`, `nombre`, `temporada`, `codigo` (6 caracteres alfanuméricos, **único**, para invitar/unirse), `visibilidad` (**siempre `privada`**: el torneo solo admite ingreso por invitación con `codigo`; no existe valor `publica`), `id_usuario_creador`, `formato` (`casual`|`incremental`|`personalizado`; mecánica de puntuación del torneo, §5), `tipo` (`competitivo`|`recreativo`; indica si el torneo gestiona **Créditos**/premios o es gratuito — **campos y reglas económicas definidos en §14**), `estado` (`activa`|`cerrada`), `id_reglas_puntuacion` (conjunto de reglas efectivas por fase del torneo, §5.4), `reglas_congeladas` (bool; `true` cuando inicia el primer partido del torneo y la puntuación queda inmutable, §5.5), `creado_en`
- **`Participaciones`** — `id`, `id_quiniela`, `id_usuario`, `roles_admin` (lista/conjunto de roles administrativos en **este** torneo: subconjunto de {`comisionado`,`tesorero`,`operador`,`vocal`}; **vacío** para un jugador puro; el creador entra con `comisionado`), `participa_como_jugador` (bool; indica si un integrante del equipo administrativo **juega además de gestionar** o **solo gestiona** —§8—; para jugadores puros siempre `true`), `unido_en`. **Tabla puente** que permite que un usuario pertenezca a **varios torneos** con **roles distintos por torneo** (§8). El **rol base de todo participante es Jugador**; `roles_admin` añade, de forma **aditiva**, los permisos del equipo administrativo. Restricciones: **un único `comisionado` por torneo**; `tesorero`/`operador`/`vocal` admiten **múltiples** usuarios. Clave lógica única (`id_quiniela` + `id_usuario`) para evitar duplicados.
- **`Equipos`** — `id`, `nombre`, `nombre_corto`, `url_escudo`
- **`Jornadas`** — `id`, `id_quiniela`, `numero`, `etiqueta` (ej. "J12"), `abre_en`, `cierra_en`, `estado` (`proxima`|`abierta`|`cerrada`|`calificada`)
- **`Partidos`** — `id`, `id_jornada`, `id_equipo_local`, `id_equipo_visitante`, `fase` (`regular`|`octavos`|`cuartos`|`semifinal`|`final`; define el multiplicador o los valores por fase de la puntuación, §5.2 y §5.3), `inicio_partido`, `fecha_limite` (**fecha/hora límite de pronóstico = `inicio_partido` − 15 minutos**, calculada server-side a partir del `inicio_partido` oficial del proveedor de API), `estado` (`programado`|`en_vivo`|`finalizado`), `goles_local`, `goles_visitante` (resultado oficial)
- **`Pronosticos`** — `id`, `id_usuario`, `id_partido`, `pronostico_local`, `pronostico_visitante`, `pronostico_resultado` (`1`|`X`|`2`), `enviado_en`, `bloqueado` (bool), `puntos_otorgados` (**valor final ya multiplicado** según el formato del torneo y la fase del partido, calculado server-side; §5.4)
- **`ReglasPuntuacion`** — `id`, `id_quiniela`, `fase` (`regular`|`octavos`|`cuartos`|`semifinal`|`final`), `puntos_resultado_correcto`, `puntos_marcador_exacto`, `multiplicador` (factor por ronda; `1` salvo en formato Incremental, donde toma `1/2/3/4/5` según §5.3.2), `congelado` (bool; `true` al iniciar el torneo, §5.5), `notas`. **Una fila por fase** con los valores **efectivos** del torneo (materializados desde §5): para `casual` e `incremental` el servidor los deriva de las constantes del algoritmo; para `personalizado` los define el **Operador** (o el Comisionado, §8) **antes del lanzamiento**. *(El antiguo `puntos_diferencia_goles` se elimina: la mecánica cerrada de la §5 solo contempla resultado correcto y marcador exacto.)*
- **`Posiciones`** (cacheable/derivable) — `id_quiniela`, `id_usuario`, `id_jornada`, `puntos_jornada`, `puntos_totales`, `posicion`, `estado_jugador` (`atrasado`|`en_riesgo`|`al_dia`, **derivado server-side**; ver §4.4.1)
- **`Usuarios`** — `id_usuario`, `proveedor_auth` (`google`|`correo`), `rol_base` (**siempre `jugador`**: el rol base de todo usuario es Jugador. **No existe rol administrativo global** — los roles del equipo administrativo, Comisionado/Tesorero/Operador/Vocal, se asignan **por torneo** en `Participaciones`, §8), `telefono`, `telefono_verificado` (bool), `correo`, `nombre_visible`, `url_avatar`, `idioma_preferido` (`es`|`en`, por defecto `es`), `tema_preferido` (`oscuro`|`claro`, por defecto `oscuro`), `creado_en`
- **`Avisos`** — `id`, `id_quiniela`, `titulo`, `cuerpo`, `publicado_en`, `id_usuario_autor`
- **`Publicidad`** (opcional) — `espacio`, `url_imagen`, `url_destino`, `activo`
- **`RegistroAuditoria`** — `marca_tiempo`, `id_usuario_actor`, `accion`, `entidad`, `id_entidad`, `meta` (sin PII)
- **Pestañas de créditos y premios** (`Pagos`, `Bolsa`, `DistribucionPremios`, `Premios`) — **se definen de forma única y cerrada en §14** (gestión de créditos y premios); aquí solo se enumeran para no dispersar la lógica económica. Aplican **solo** a torneos `tipo=competitivo`.

**Fuente de datos de partidos (en vivo, pasados y futuros):** La sección "En Vivo", así como el listado de partidos pasados y futuros de cada jornada (`PartidosAPronosticar`, `BarraJornadas`), **debe actualizarse únicamente mediante datos obtenidos de un proveedor externo de API de resultados deportivos en vivo** (p. ej. API-Football, SportRadar, o equivalente), y no mediante captura manual del **Operador** (§8). El backend (Apps Script o proxy serverless) debe consumir dicho API de forma periódica o mediante webhook, normalizar la respuesta al esquema de `Partidos` (`estado`, `goles_local`, `goles_visitante`, `inicio_partido`), y persistir/actualizar los registros en Google Sheets. La clave de API del proveedor debe mantenerse **únicamente del lado del servidor** (nunca en variables `VITE_*` ni expuesta al cliente), y el cálculo de puntos (`calificarJornada`) debe ejecutarse solo cuando el proveedor marque el partido como `finalizado`, manteniendo el principio de que el cliente nunca es fuente de verdad. Además, el proveedor de API debe exponer la **tabla final de la fase regular** de la liga y el **equipo campeón** del torneo real; estos datos alimentan el **Premio al Campeón** definido en **§14**.

**Cálculos:** los puntos de cada `Pronostico` se calculan **server-side** con el **algoritmo canónico de la §5.4**, analizando el **marcador pronosticado** frente al resultado oficial de `Partidos` y aplicando el **formato** del torneo (Casual/Incremental/Personalizado) y la **fase** del partido sobre las `ReglasPuntuacion` efectivas y congeladas (§5.5). `Posiciones` se recalcula al calificar una jornada (`Jornadas.estado = calificada`). Los contadores del UI (pendientes, cierran hoy) se derivan de `Pronosticos` + `Partidos.fecha_limite`. El **estado de jugador** (`atrasado`|`en_riesgo`|`al_dia`, §4.4.1) también se deriva de `Pronosticos` + `Partidos.fecha_limite` (= `inicio_partido` − 15 min) contra la **hora del servidor**; al depender de una ventana de 24 h relativa al momento de consulta, se calcula al leer o se cachea con TTL corto.

---

## 7. Capa de acceso a datos

**Lectura (GET):**
- `obtenerMisQuinielas()` — torneos en los que participa el usuario autenticado (desde `Participaciones`), con su rol por torneo.
- `obtenerJornadaActiva(idQuiniela)` — jornada abierta y sus partidos.
- `obtenerPartidosJornada(idJornada)` — partidos con su estado y fecha límite (sincronizados desde el proveedor de API externo, ver sección 6).
- `obtenerMisPronosticos(idJornada)` — pronósticos del usuario autenticado para esa jornada.
- `obtenerTablaPosiciones(idQuiniela, idJornada?)` — tabla de posiciones (general o por jornada), incluyendo el **`estado_jugador`** (`atrasado`|`en_riesgo`|`al_dia`, §4.4.1) de cada participante, calculado server-side.
- `obtenerEstadoJugadores(idQuiniela)` — estado de cumplimiento (`atrasado`|`en_riesgo`|`al_dia`) de cada participante del torneo activo, derivado server-side de los partidos sin pronosticar y la `fecha_limite` de cada partido (§4.4.1).
- `obtenerAvisos(idQuiniela)` / `obtenerReglamento(idQuiniela)` (este último incluye el **formato** del torneo y los puntos por **resultado correcto / marcador exacto** de cada fase, §5).

**Escritura (POST → Apps Script con validación):**
- `crearQuiniela(nombre, temporada, formato, reglas?, tipo, costo_inscripcion_creditos?)` — crea un torneo con su **formato** (`casual`|`incremental`|`personalizado`, §5); para `personalizado`, `reglas` trae los puntos por **resultado** y **marcador exacto** **por fase** (§5.3.3), que se validan y **congelan al lanzamiento** (§5.5). El **servidor genera el `codigo` único de 6 caracteres alfanuméricos** (con verificación de colisión) y registra al creador en `Participaciones` como **Comisionado** (`roles_admin=[comisionado]`), permitiéndole indicar si **participará como jugador** o **solo gestionará** (`participa_como_jugador`, §8). La firma incluye también el **`tipo`** del torneo (`competitivo`|`recreativo`) y, si es `competitivo`, el **`costo_inscripcion_creditos`**; su captura, validación y mecánica económica se definen de forma única en **§14.7** (esta sección solo los referencia). Devuelve el código.
- `unirseQuiniela(codigo)` — valida el `codigo` server-side; si existe y el usuario no participa aún, crea la `Participacion` como **Jugador** (`roles_admin` vacío, `participa_como_jugador=true`). Rechaza códigos inválidos/inexistentes y la doble unión.
- `expulsarJugador(idQuiniela, idUsuario)` — **solo el Comisionado**; expulsa a cualquier participante en **cualquier momento** del torneo. **No existe abandono voluntario:** un participante **no puede salir por iniciativa propia** (§8); la única baja posible es la expulsión por parte del Comisionado.
- `asignarRol(idQuiniela, idUsuario, rol)` / `removerRol(idQuiniela, idUsuario, rol)` — **solo el Comisionado**; asigna o retira `tesorero`/`operador`/`vocal` (cada puesto admite **múltiples** usuarios, §8).
- `transferirComisionado(idQuiniela, idUsuarioDestino)` — **solo el Comisionado**; transfiere el rol de **Comisionado** (único por torneo) a otro participante.
- `configurarParticipacion(idQuiniela, participaComoJugador)` — un integrante del equipo administrativo define si **juega además de gestionar** o **solo gestiona** (`participa_como_jugador`, §8).
- `enviarPronostico(idPartido, pronosticoLocal, pronosticoVisitante, pronosticoResultado)` — **rechaza si ya pasó la `fecha_limite` del partido** (es decir, si faltan **menos de 15 minutos** para el inicio o el partido ya comenzó) o si la jornada no está abierta; el `id_usuario` se deriva del token, no del cuerpo de la solicitud. Verifica además que el usuario **participe** en el torneo del partido.
- `guardarUsuario(...)` — perfil del usuario autenticado, incluidas sus preferencias `idioma_preferido` (`es`|`en`) y `tema_preferido` (`oscuro`|`claro`).
- **(equipo administrativo del torneo — permisos por rol, §8)** Gestión **operativa** (**Operador**, o **Comisionado** en su defecto): `crearJornada(...)`, `guardarPartido(...)`, `definirReglasPuntuacion(idQuiniela, reglasPorFase)` (**solo** formato `personalizado` y **solo antes** del lanzamiento/primer partido, §5.3.3 y §5.5), `calificarJornada(idJornada)` (dispara el **cálculo de puntos según la §5.4** —formato + fase— y actualiza `Posiciones`; se ejecuta automáticamente cuando el proveedor de API marca el partido como `finalizado`). Gestión de **comunicación** (**Vocal**, o **Comisionado** en su defecto): `publicarAviso(...)`. Gestión **económica** (**Tesorero**, o **Comisionado** en su defecto): confirmación de inscripciones, consolidación de la **Bolsa Única** y **distribución de premios** (incluido el **Premio al Campeón**) — **endpoints, datos y reglas definidos exclusivamente en §14** (créditos y premios). Gestión de **personas y roles** (**solo Comisionado**): `asignarRol`/`removerRol`, `transferirComisionado`, `expulsarJugador`. Cada operación exige el **rol administrativo correspondiente en ese torneo** (verificado server-side, **principio de menor privilegio**); si no se ha asignado a nadie como Tesorero/Operador/Vocal, el **Comisionado asume automáticamente** esos permisos (§8).

**Estados UI:** `cargando` (skeleton), `vacio` ("Sin partidos en esta jornada"), `bloqueado` ("Pronóstico cerrado"), `error` (reintentar). Cachear lecturas por (quiniela + jornada).

---

## 8. Autenticación y roles (flujo) — modelo de roles canónico (fuente única)

> **Fuente única de roles.** Esta sección es la **única** definición del modelo de roles del producto. Ninguna otra sección redefine roles, permisos ni cardinalidad: el *Resumen* (§1), la *UI/UX* (§4), la *Mecánica* (§5), el *Modelo de datos* (§6), la *Capa de acceso a datos* (§7), el *Checklist* (§12) y la *Seguridad* (§13) **se subordinan y remiten** a este modelo.

1. **Disparador:** tap en perfil (encabezado) o intento de **enviar un pronóstico** sin sesión → abre `<ModalAutenticacion>`.
2. **Principal — Google:** botón "Continuar con Google" (Google Identity Services). Tras el login, el **número de celular es obligatorio por esta vía** y se valida por **OTP/SMS**: `telefono_verificado=true` (fijado **solo server-side**, §13 A07) es **requisito para capturar pronósticos**. Mientras el teléfono no esté verificado, la sesión queda **limitada** (puede consultar jornadas/ranking, **no** pronosticar).
3. **Alternativa — Correo:** botón secundario "Continuar con correo" (enlace mágico o correo+contraseña vía Apps Script/serverless). Por esta vía **no se exige número de celular** (`telefono`/`telefono_verificado` quedan vacíos/`false`); es la alternativa para quien no use Google. *(Si el organizador decidiera exigir teléfono también aquí, es una decisión de producto: ajústese este paso y `Usuarios` en §6.)*
4. **Modelo de roles (definición canónica y única del producto).** Toda la lógica de roles vive aquí; el resto de secciones se subordina a este modelo.

   **4.1 Rol base.** El rol base de **todo** usuario, en **todo** torneo, es **Jugador** (juega pronósticos y consulta ranking/avisos). **No existe rol administrativo global:** cualquier privilegio de gestión es **aditivo** y se concede **por torneo**.

   **4.2 Equipo administrativo (cuatro roles, por torneo).** Un torneo puede tener, además de jugadores, un **equipo administrativo** con estos roles y responsabilidades (base del **principio de menor privilegio**, §13):
   - **Comisionado** — máxima autoridad del torneo. **Único por torneo.** Es el **único** facultado para **asignar y transferir** roles administrativos y para **expulsar jugadores**. Supervisa el torneo; cuando un puesto (Tesorero/Operador/Vocal) **no** está asignado a nadie, **asume automáticamente** sus responsabilidades y permisos. Se asigna **por defecto al creador** del torneo.
   - **Tesorero** — gestión **económica** del torneo (en **Créditos**): confirma inscripciones (`preinscrito`→`inscrito`), consolida la **Bolsa Única** y ejecuta la **distribución de premios**, incluido el **Premio al Campeón**. **Flujos, paneles, endpoints y datos detallados de forma única en §14.** No gestiona jornadas/partidos, avisos ni roles.
   - **Operador** — gestión **operativa del juego**: jornadas y partidos, configuración del **formato y reglas de puntuación** (antes del lanzamiento, §5.5), supervisión de la sincronización de resultados y disparo/recálculo de la calificación (§5.4). No gestiona créditos/premios, avisos ni roles.
   - **Vocal** — **comunicación** con los participantes: publicación de avisos/comunicados y noticias del torneo. No gestiona partidos, créditos ni roles.

   **4.3 Cardinalidad.** **Exactamente un Comisionado** por torneo. **Tesorero, Operador y Vocal** admiten **múltiples** usuarios cada uno, y un mismo usuario puede acumular varios de estos roles en un torneo. **Regla de absorción:** si no se designan administradores adicionales, **el Comisionado ejerce todos los roles restantes** (Tesorero + Operador + Vocal) de forma automática.

   **4.4 Granularidad por torneo.** Los roles se asignan **por torneo, nunca de forma global** (`Participaciones`, §6). Un usuario puede ser **Comisionado/Tesorero/Operador/Vocal en un torneo y, a la vez, Jugador puro en otro**, sin que un rol en un torneo otorgue privilegio alguno en los demás.

   **4.5 Administrador jugador vs. solo-gestión.** Cada integrante del equipo administrativo elige, **por torneo**, si **participa activamente en el juego** (gestiona **y** juega) o si **solo ejerce funciones de gestión, sin posibilidad de jugar** (`participa_como_jugador`, §6). Un administrador en modo **solo-gestión** **no** captura pronósticos ni aparece en el ranking de ese torneo.

   **4.6 Permanencia y expulsión (control del Comisionado).** El **Comisionado** puede **expulsar a cualquier jugador en cualquier momento** del torneo. Los participantes **no pueden abandonar el torneo por iniciativa propia**: no existe operación de salida voluntaria (§7); la única baja posible es por **expulsión** del Comisionado. La **transferencia del rol de Comisionado** a otro participante es la vía para que el creador deje de ejercerlo.

   **4.7 Menor privilegio (autorización).** La **visualización y modificación** de las funciones avanzadas (paneles, endpoints y acciones de gestión) se restringen **exclusivamente al equipo administrativo**, y dentro de él **según las responsabilidades de cada rol** (§4.2). Toda autorización se verifica **server-side por torneo** desde `Participaciones`; ocultar controles en el UI es solo cosmético (§13 A01). **Denegar por defecto.**
5. **Sesión:** reflejar avatar+nombre en el encabezado. Pronósticos, posición y pendientes ligados al `id_usuario`.
6. **Seguridad:** toda escritura (incluido enviar pronóstico) pasa por Apps Script validando el `id_token` de Google, la **fecha límite** y el **rol por torneo** (§8.4). El cliente solo conoce el **ID de Cliente** público.

---

## 9. Estructura de carpetas sugerida

```
quiniela-liga/
├── CLAUDE.md                 # este archivo
├── index.html                # ★ ÚNICO ARCHIVO DEL CLIENTE (SPA estricta): contiene TODO
│                             #   <head>: meta TDM opt-out (tdm-reservation) + robots noai/noindex
│                             #           + aviso de copyright + CSP (secciones 13 y 14)
│                             #   <style>: tokens de diseño + temas oscuro/claro (sección 3, embebidos)
│                             #   <body>: marco móvil + nodo raíz de montaje (p. ej. <div id="app">)
│                             #   <script>: TODA la lógica (vistas, modales, i18n, datos, auth, estado)
│
│   ── Organización LÓGICA interna del <script> (NO son archivos: son módulos/secciones en memoria) ──
│      • Tema:               ProveedorTema, usarTema  (oscuro|claro; aplica data-tema)
│      • Idioma (i18n §9.1): RECURSOS.es / RECURSOS.en (objetos embebidos), t(), ProveedorIdioma, usarIdioma
│      • Vistas/Componentes: EncabezadoApp, SelectorQuiniela (lista + crear + unirse), BarraJornadas,
│                            BarraJornadaActual, PanelMiQuiniela, PartidosAPronosticar, FilaPartido,
│                            TablaPosiciones, EstadoJugadores (§4.4.1), EspacioPublicidad, NavInferior
│      • Modales/Selectores: ModalAutenticacion, ModalCrearQuiniela (muestra el código generado),
│                            ModalUnirseQuiniela (código de 6 caracteres), SelectorTema, SelectorIdioma
│      • Equipo administrativo (por torneo, §8): PanelAdmin (vistas por rol: Comisionado/Tesorero/Operador/Vocal),
│                            GestorRoles (asignar/transferir, solo Comisionado), GestorJornadas, SincronizacionResultados,
│                            PanelTesoreria + CalculadoraDistribucion (créditos/premios — solo Competitivos, §14)
│      • Datos:              api (llamadas a Apps Script / Sheets), auth (Google Identity + correo + roles por torneo, §8)
│      • Hooks/estado:       usarMisQuinielas, usarJornada, usarPronosticos, usarPosiciones,
│                            usarEstadoJugadores (§4.4.1)
│      • Enrutador en memoria: conmuta Jugar / En Vivo / Ranking / Avisos / Reglas SIN rutas ni HTML extra
│
│   (Los escudos provienen del proveedor de API vía `url_escudo`; si se requiere alguno local,
│    se incrusta como data URI dentro del index.html para no romper el archivo único.)
│
├── LICENSE                   # licencia restrictiva — Todos los derechos reservados (sección 15)
├── package.json
├── vite.config.js            # build single-file: inline total + minify + SIN sourcemaps + sin comentarios
│                             #   (secciones 11 y 15)
├── .env                      # NO commitear (sección 10)
├── .github/
│   └── workflows/
│       └── deploy.yml        # build single-file + deploy a GitHub Pages
├── public/                   # assets servidos en la RAÍZ del sitio (no son vistas HTML)
│   └── robots.txt            # Disallow a crawlers no autorizados + noai + noindex (sección 15)
└── apps-script/
    └── Codigo.gs             # BACKEND aparte (Google Apps Script) — NO forma parte del index.html:
                              #   auth, fechas límite, sincronización con API de resultados,
                              #   cálculo de puntos, posiciones, generación de códigos de torneo
                              #   créditos: pagos/Bolsa/distribución y Premio al Campeón (solo Competitivos, §14)
```

### 9.1 Recursos de idioma (i18n)
- Dos diccionarios planos como **objetos JS embebidos** en el `index.html`: `RECURSOS.es` (por defecto) y `RECURSOS.en`, con **las mismas claves**. Toda cadena visible vive aquí; los componentes/vistas solo referencian claves vía `t('clave')`. **No se permite texto incrustado** en el marcado generado. (El requisito de externalizar el texto se mantiene; solo cambia el soporte: objetos en memoria en lugar de archivos `.json` sueltos, para honrar el archivo único.)
- Mantener `RECURSOS.es` como fuente de verdad; cada clave nueva debe existir en ambos objetos (validar paridad de claves en build/CI si es viable).

### 9.2 Persistencia de preferencias (tema e idioma)
- **Tema** e **idioma** se guardan en `localStorage` (`tema` = `oscuro|claro`, `idioma` = `es|en`). Estas **no son credenciales**, por lo que `localStorage` es aceptable aquí (a diferencia de los tokens, sección 13).
- Al iniciar: leer preferencia guardada → si no existe, usar **oscuro** y **es** por defecto (idioma puede inicializarse desde `navigator.language` solo si es `en`, pero el valor por defecto del producto es `es`).
- Si el usuario está autenticado, sincronizar la preferencia con `Usuarios.idioma_preferido`/`tema_preferido` para portabilidad entre dispositivos.

---

## 10. Variables de entorno / configuración

Crear `.env` (NO commitear) y exponer solo lo público vía Vite (prefijo `VITE_`):

```
VITE_GOOGLE_CLIENT_ID=xxxxxxxx.apps.googleusercontent.com
VITE_SHEETS_API_ENDPOINT=https://script.google.com/macros/s/XXX/exec
```

> En el build single-file, estas variables `VITE_*` quedan **incrustadas dentro del `index.html`** resultante y son **públicas por diseño** (cualquiera puede leerlas en el HTML servido). **Nunca** colocar secretos aquí.

> **Sin lectura directa de Google Sheets desde el cliente (no existe `VITE_SHEET_ID`).** El cliente **no** recibe el ID de la hoja ni usa la API de Google Sheets directamente: **toda** lectura pasa por el endpoint de Apps Script (`VITE_SHEETS_API_ENDPOINT`), que es el **único** que aplica validación de `id_token`, **pertenencia a torneo** y **fecha límite** (§7 y §13). Exponer un `SHEET_ID` público permitiría que cualquier usuario leyera pestañas como `Pronosticos` de otros jugadores **antes del cierre** y accediera a torneos donde **no participa**, rompiendo el anti-trampa y el aislamiento por torneo (§13). Por eso la lectura directa de Sheets **queda prohibida** y la variable correspondiente **no** se define.

Secretos de escritura (cuenta de servicio, clave del proveedor de API de resultados en vivo, claves de SMS, etc.) viven **solo** dentro de Apps Script, nunca en el repo del frontend.

---

## 11. Build y despliegue a GitHub Pages

`vite.config.js` debe fijar `base` al nombre del repo **y producir un único `index.html` autocontenido** (inline total, sin sourcemaps, minificado y sin comentarios — ver sección 15):

```js
import { defineConfig } from 'vite'
import { viteSingleFile } from 'vite-plugin-singlefile'

export default defineConfig({
  base: '/quiniela-liga/',
  plugins: [viteSingleFile()],          // inlina JS + CSS en un único index.html
  build: {
    target: 'esnext',
    sourcemap: false,                    // PI: SIN source maps en producción (§15)
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

## 12. Checklist de paridad visual y funcional (criterio de aceptación)

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
- [ ] Los puntos se calculan server-side con el **algoritmo de la §5.4** según el **formato** del torneo (Casual/Incremental/Personalizado) y la **fase** del partido, sobre las `ReglasPuntuacion` **congeladas** (§5.5), y se reflejan en el ranking.
- [ ] El torneo tiene un **formato** definido (Casual / Incremental / Personalizado); en **Personalizado**, los puntos por fase se fijaron **antes del lanzamiento** y quedaron **congelados** al iniciar el primer partido (§5).
- [ ] `TablaPosiciones` muestra la tabla de posiciones (general y por jornada) **del torneo activo**.
- [ ] Se muestra el **estado de cada jugador** —**Atrasado**, **En Riesgo** o **Al día**— en `PanelMiQuiniela` (propio) y en `TablaPosiciones` (todos), calculado server-side según §4.4.1.
- [ ] **Roles por torneo (§8):** el rol base es **Jugador**; el **equipo administrativo** (Comisionado/Tesorero/Operador/Vocal) accede al panel y endpoints de administración **según su rol** (**menor privilegio**). Un único **Comisionado** por torneo (creador por defecto), único que **asigna/transfiere** roles administrativos y **expulsa** jugadores; **sin abandono voluntario**. Si no hay Tesorero/Operador/Vocal, el Comisionado **asume** esos permisos. Los administradores pueden optar por **solo gestionar** (sin jugar).
- [ ] **Créditos y premios (§14):** el **tipo de torneo** (**Competitivo**/**Recreativo**) se elige al crear; en Competitivo, el **Tesorero** confirma inscripciones y la app consolida la **Bolsa Única en Créditos**; la **calculadora de distribución** reparte la Bolsa en **fases regulares**, **eliminación directa**, **Campeón** y **retención administrativa** (porcentajes o valores fijos, cuadrando el total); el **Premio al Campeón** se asigna por **posición cruzada** (la posición del campeón real en la fase regular ↔ el jugador en esa misma posición de la quiniela); en **toda** la UI el valor se denomina **«Créditos»**. Lista completa en §14.

---

## 13. Seguridad — OWASP Top 10 (2021) con controles concretos

> **Nivel de datos:** se manejan **datos personales** (nombre, correo, **teléfono verificado**) y, además, **datos de integridad competitiva** (pronósticos, puntos, ranking) con potencial valor económico si el torneo es **Competitivo** (créditos/premios, §14). Tratar teléfono/correo como **PII** y los pronósticos/puntos como datos de **integridad crítica**.

### Principio rector de arquitectura
Con **GitHub Pages + Google Sheets**, el frontend es **público y no confiable**: cualquiera puede leer su código, sus variables `VITE_*` y llamar directamente al endpoint. La **única frontera de seguridad real es Apps Script (o el proxy serverless)**. Regla absoluta: **toda autorización, validación, control de fecha límite y cálculo de puntos ocurre del lado del servidor; el cliente nunca es fuente de verdad.**

### Riesgo específico de quiniela (integridad del juego) — máxima prioridad
- **Anti-trampa de fecha límite (15 min antes del inicio):** `enviarPronostico` debe **rechazar server-side** cualquier pronóstico recibido después de la `fecha_limite` del partido, definida como **`inicio_partido` − 15 minutos** y evaluada con la **hora del servidor** (nunca la del cliente). La `fecha_limite` se **deriva en el servidor** del `inicio_partido` oficial del proveedor de API, de modo que el cliente no pueda alterarla. Un jugador no puede pronosticar un partido cuyo cierre (15 min antes del inicio) ya ocurrió, ni uno ya iniciado/terminado.
- **Inmutabilidad tras cierre:** una vez bloqueado (`bloqueado=true`) o calificado, el pronóstico **no puede modificarse**, ni siquiera por el mismo usuario.
- **No filtrar pronósticos ajenos antes del cierre:** los endpoints de lectura **no** deben exponer los pronósticos de otros participantes mientras la jornada esté abierta (evita copiar). Solo se revelan tras la fecha límite.
- **Resultados y cálculo de puntos solo server-side:** los resultados oficiales se reciben únicamente desde el proveedor de API verificado (sección 6) y `calificarJornada` es una operación interna del servidor; los puntos nunca se aceptan desde el cliente.
- **Idempotencia y auditoría:** calificar una jornada debe ser idempotente y quedar registrado en `RegistroAuditoria` (quién/qué proceso, cuándo, qué cambió).
- **Congelamiento de la puntuación (transparencia/legalidad):** el **formato** y las **reglas por fase** (`P_RES`/`P_EXACTO`, multiplicadores) se **validan y congelan server-side al lanzar el torneo** (al iniciar el primer partido, §5.5). En **Personalizado** deben fijarse **antes** del lanzamiento; el backend **rechaza** toda edición posterior. El cliente nunca altera `formato`, `fase` ni los puntos: el cálculo aplica el **algoritmo canónico de la §5.4** únicamente en el servidor.

### Riesgo de multi-torneo y códigos de invitación
- **Privacidad de torneos (solo por invitación):** todos los torneos son **privados**; el backend **no** expone ningún endpoint de listado o descubrimiento de torneos ajenos, y `obtenerMisQuinielas` devuelve únicamente torneos donde el usuario ya tiene `Participacion`. La **única** vía de ingreso es `unirseQuiniela(codigo)` con un código válido recibido por invitación.
- **Generación del código server-side:** el `codigo` de 6 caracteres alfanuméricos se genera **solo en el servidor** con aleatoriedad adecuada, verificando **colisión** contra `Quinielas.codigo` y reintentando hasta obtener uno único. El cliente nunca propone ni fija el código. (Opcional: excluir caracteres ambiguos como `0/O`, `1/I/l` para evitar errores de tecleo.)
- **Anti-enumeración/fuerza bruta de códigos:** `unirseQuiniela` debe tener **rate limiting estricto** por usuario/IP (con `CacheService`/`PropertiesService`) y backoff ante intentos fallidos repetidos, para impedir adivinar códigos por fuerza bruta. Registrar ráfagas de intentos fallidos en `RegistroAuditoria` y alertar.
- **Validación de formato:** rechazar cualquier `codigo` que no cumpla `^[A-Za-z0-9]{6}$` antes de tocar datos (lista blanca de caracteres).
- **Autorización por pertenencia a torneo:** toda lectura/escritura se restringe a torneos donde el usuario tiene una `Participacion`. Un usuario **no** puede leer jornadas, pronósticos, ranking ni avisos de torneos a los que no pertenece. El rol admin se evalúa **por torneo** (`Participaciones.roles_admin`, §8), nunca global.
- **Aislamiento entre torneos:** el `id_quiniela` objetivo se valida server-side contra la pertenencia del usuario; no confiar en el `id_quiniela` enviado por el cliente sin verificar membresía.

### A01 — Broken Access Control
- El cliente **nunca** envía `id_usuario` confiable. El servidor deriva la identidad **del `id_token` verificado**; los **roles administrativos** se derivan **por torneo** desde `Participaciones` (§8), nunca de un rol enviado por el cliente.
- Autorización por propiedad de recurso: un jugador solo lee/escribe **sus** `Pronosticos` y su `Usuarios`, y **solo dentro de torneos donde participa**. Las operaciones de gestión exigen el **rol administrativo correspondiente en ese torneo** (Operador para jornadas/partidos/calificación; Vocal para avisos; Tesorero para **créditos/premios** (§14); **solo Comisionado** para asignar/transferir roles y **expulsar jugadores**), verificado server-side (**menor privilegio**, §8).
- **Denegar por defecto** (HTTP 403). No exponer endpoints de gestión en la ruta pública sin verificación del **rol administrativo correspondiente** (§8).
- No confiar en ocultar el panel admin en el UI como medida de acceso (es solo cosmético): el **menor privilegio** se aplica **server-side** por rol (§8).

### A02 — Cryptographic Failures (datos sensibles)
- **HTTPS obligatorio** extremo a extremo; nunca contenido mixto `http://`.
- **PII en reposo:** teléfono y correo normalizados y minimizados; considerar **hash** (SHA-256 con sal) del teléfono para búsquedas si el negocio no requiere el número en claro.
- **Tokens:** no persistir el `id_token` en `localStorage` (riesgo XSS). Mantener en memoria; renovar vía Google Identity. Sesión propia con cookie `HttpOnly`+`Secure`+`SameSite=Strict` si el proxy lo permite.
- No registrar PII ni tokens en logs ni en mensajes de error.

### A03 — Injection (incluye Formula/CSV Injection en Sheets)
- **CSV/Formula injection (crítico aquí):** todo valor de usuario escrito en celda que empiece con `=`, `+`, `-`, `@`, tab o CR debe **sanitizarse** (anteponer `'`) o rechazarse. Aplica a `nombre_visible`, **nombre del torneo**, correo y cualquier texto libre (avisos, notas).
- **Validación estricta server-side:** pronósticos numéricos no negativos y dentro de rango razonable; `pronostico_resultado` ∈ {`1`,`X`,`2`}; `id_partido`/`id_jornada` deben existir y pertenecer a la jornada abierta (lista blanca); **`codigo` de torneo ∈ `^[A-Za-z0-9]{6}$` y debe existir**; correo válido; teléfono E.164; `formato` ∈ {`casual`,`incremental`,`personalizado`}; `fase` ∈ {`regular`,`octavos`,`cuartos`,`semifinal`,`final`}; en `personalizado`, los puntos por fase son enteros ≥ 0 y deben quedar definidos **antes del lanzamiento** (§5.5).
- Renderizar datos de usuario con `textContent`, nunca `innerHTML`. **i18n:** interpolar variables en las cadenas traducidas de forma segura (sin `innerHTML`); las traducciones no deben permitir inyección de marcado.

### A04 — Insecure Design
- **Rate limiting / anti-abuso** en el endpoint (por usuario/IP con `CacheService`/`PropertiesService`), y **especialmente** en envío de OTP/SMS (anti-bombing y control de costos) y en **`unirseQuiniela`** (adivinación de códigos por fuerza bruta).
- **Modelo de amenazas documentado:** anónimo en internet, jugador malicioso (pronosticar tarde, editar tras cierre, ver pronósticos ajenos, autoasignarse puntos o **roles administrativos**, **autoasignarse el rol de Comisionado o expulsar a otros sin serlo**, **intentar abandonar el torneo pese a no estar permitido**, **adivinar códigos de torneo o acceder a torneos donde no participa**), abuso de SMS, scraping del ranking, manipulación de la sincronización con el proveedor de API de resultados.
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
- **Teléfono verificado:** `telefono_verificado=true` solo tras validar OTP correcto en ventana; el cliente no puede establecer ese valor. Igual para los **roles administrativos**: el cliente **no** puede establecer `roles_admin` ni autoasignarse Comisionado; los roles se asignan **solo** server-side por el Comisionado (§8).
- Alternativa por correo: enlace mágico de un solo uso/expirable, o contraseña con hashing fuerte (bcrypt/scrypt/argon2) — **nunca** PII/credenciales en claro en Sheets.
- Sesiones con expiración; logout invalida sesión server-side.

### A08 — Software & Data Integrity Failures
- Dependencias con **lockfile** (`npm ci`) en CI para builds reproducibles.
- Scripts de terceros (Google Identity, publicidad) desde orígenes oficiales; usar **SRI** donde la fuente sea estática.
- GitHub Actions con permisos mínimos (sección 11); acciones de terceros pinneadas por SHA en contextos sensibles.
- **Integridad del juego:** cualquier cambio a resultados oficiales o puntos queda en `RegistroAuditoria`; recalificar es idempotente y trazable.
- **Integridad de la sincronización de resultados:** validar la firma/origen de las respuestas del proveedor de API de resultados (o el secreto del webhook) antes de actualizar `Partidos`.

### A09 — Security Logging & Monitoring Failures
- Registrar en `RegistroAuditoria`/logs (sin PII): auth fallida, OTP fallido, accesos 403, intentos de pronóstico fuera de plazo, ediciones tras cierre, cambios de resultado/puntos, picos de rate limit, fallos de sincronización con el proveedor de API de resultados.
- Alertar ante anomalías (muchos OTP a un número/IP; ráfagas de pronósticos justo en la fecha límite; caída del proveedor de API de resultados).

### A10 — Server-Side Request Forgery (SSRF)
- Bajo en esta arquitectura, pero si el servidor hace fetch a URLs derivadas de input (`url_avatar`, `url_escudo`, `url_destino` de publicidad) o a la API del proveedor de resultados, **validar contra lista blanca de dominios** y no seguir redirecciones a rangos internos.

---

### Privacidad / cumplimiento (teléfono + PII + posible premio en Créditos, §14)
- **Minimización de datos** y justificación de por qué se pide el teléfono.
- **Consentimiento** explícito para SMS y aviso de privacidad accesible.
- **Derecho de borrado:** operación para eliminar la cuenta y su PII (conservando, si aplica, registros mínimos de integridad del juego de forma anonimizada).
- Si el torneo es **Competitivo** (gestiona **Créditos**/premios, §14), revisar la legalidad local de apuestas/sorteos/rifas antes de operar; esto excede lo técnico. La terminología «Créditos» (§14) **reduce, pero no elimina**, el riesgo de interpretación.

---

### Checklist de seguridad (criterio de aceptación)
- [ ] **Fecha límite server-side (15 min antes del inicio):** se rechaza todo pronóstico posterior a `fecha_limite = inicio_partido − 15 min` usando la hora del servidor; la `fecha_limite` se **deriva server-side** del `inicio_partido` oficial.
- [ ] Pronósticos inmutables tras cierre/calificación; sin edición posterior.
- [ ] Pronósticos ajenos **no** visibles antes de la fecha límite.
- [ ] Resultados oficiales sincronizados **solo** desde el proveedor de API verificado; cálculo de puntos **solo** server-side; nunca aceptados del cliente.
- [ ] **Puntuación por formato (§5):** `formato` y reglas por fase validados y **congelados al lanzamiento**; cálculo con el **algoritmo de la §5.4** server-side; en `personalizado`, los valores se fijan **antes** del primer partido y son **inmutables** después.
- [ ] Identidad derivada del **`id_token` verificado** (firma, `aud`, `iss`, `exp`); los **roles administrativos se evalúan por torneo** desde `Participaciones` (§8); el cliente nunca envía `id_usuario`/roles confiables.
- [ ] Autorización por propiedad de recurso **y por pertenencia a torneo**; denegar por defecto; endpoints de gestión protegidos por el **rol administrativo correspondiente** en ese torneo (**menor privilegio**, §8); **solo el Comisionado** asigna/transfiere roles y **expulsa** jugadores; **sin abandono voluntario**.
- [ ] **Códigos de torneo:** generados server-side, únicos (sin colisión), validados con `^[A-Za-z0-9]{6}$`; `unirseQuiniela` con rate limiting/backoff anti-fuerza bruta. **Torneos privados:** sin endpoint de listado/descubrimiento; el único ingreso es por invitación con el código.
- [ ] **Sanitización anti-formula injection** en toda celda con input de usuario (`= + - @`).
- [ ] Validación server-side: pronóstico en rango, `resultado` ∈ {1,X,2}, ids en lista blanca de la jornada abierta, correo, teléfono E.164.
- [ ] CORS al origen de Pages; CSP en `index.html`; orígenes del ID de Cliente limitados.
- [ ] Rate limiting en endpoint y en OTP/SMS; OTP de un solo uso/expirable; `telefono_verificado` y los **roles administrativos** solo server-side.
- [ ] Tokens fuera de `localStorage`; HTTPS estricto; sin contenido mixto.
- [ ] Publicidad en `<iframe sandbox>`; `npm ci` + `npm audit` en CI; Dependabot activo.
- [ ] `RegistroAuditoria` de eventos de integridad y seguridad sin PII; borrado de cuenta disponible.
- [ ] Sincronización con el proveedor de API de resultados autenticada/validada; clave de API solo en el servidor.
- [ ] **Créditos y premios (torneos Competitivos):** confirmación de inscripción **solo Tesorero**; **Bolsa** y **distribución** calculadas/validadas server-side (la distribución debe cuadrar al 100% / total de la Bolsa antes de congelarse); **Premio al Campeón** resuelto server-side desde la **tabla final verificada** del proveedor de API; cambios de estatus de pago y distribución en `RegistroAuditoria`. **Controles detallados en §14.**

---

## 14. Gestión de Créditos y Premios (fuente única)

> **Fuente única de lo económico.** Esta sección es la **única** definición de **todo** lo relacionado con créditos y recompensas del producto: **terminología**, **tipo de torneo**, **costos de inscripción**, **pagos**, **Bolsa Única**, **distribución** y **premios** (incluido el **Premio al Campeón**). Ninguna otra sección define reglas económicas: el *Resumen* (§1), la *UI/UX* (§4), la *Mecánica de puntos* (§5), el *Modelo de datos* (§6), la *Capa de acceso a datos* (§7), el *Modelo de roles* (§8), la *Estructura* (§9), el *Checklist* (§12) y la *Seguridad* (§13) **solo la referencian y se subordinan a ella**. Toda la lógica de créditos/premios se concentra **aquí**, para garantizar un control absoluto y ordenado del flujo.
>
> **Ámbito de aplicación.** Esta sección aplica **exclusivamente** a torneos con `tipo = competitivo`. Los torneos `tipo = recreativo` **no** muestran ni gestionan ningún elemento económico (sin costo, sin Bolsa, sin premios, sin paneles ni vistas de créditos).

### 14.1 Terminología obligatoria («Créditos») — prevención de contingencias legales

- **Regla absoluta de UI.** En **todo** el texto visible de la aplicación, cualquier **valor, saldo, costo, cuota, monto, bolsa o premio** se denomina **única y exclusivamente «Créditos»** (singular: «Crédito»).
- **Palabras prohibidas en el texto visible:** «dinero», «pesos», «dólares», «efectivo» (y cualquier otra moneda), así como **símbolos de moneda** (`$`, `€`, `£`, etc.) usados para denominar valor dentro de la app. El objetivo es **prevenir contingencias legales** y evitar que el producto se interprete como una plataforma de apuestas con dinero real.
- **i18n (§9.1).** Todas las cadenas económicas viven en `RECURSOS.es`/`RECURSOS.en` y usan «Créditos»/«Credits». **No** se escriben montos con prefijo/sufijo de moneda; el formato es `{n} Créditos` (es) / `{n} Credits` (en).
- **Identificadores de código (no afectados).** La prohibición es **solo de texto visible**. Los nombres de variables, funciones, columnas y enums **siguen en español** (§0): p. ej. `costo_inscripcion_creditos`, `monto_creditos`, `Bolsa`. El término técnico de campo puede contener «creditos»; lo prohibido es **mostrar** moneda real al usuario.
- **Canales de pago vs. denominación de valor (conciliación).** El cobro es **externo** (§14.4) y el **Tesorero** registra **cómo** llegó cada pago. Para **no** romper la regla de terminología, las etiquetas de **canal** se nombran como **medio**, nunca como moneda: **«Transferencia»**, **«Depósito»** y **«Pago presencial»** (este último cubre la entrega directa/efectivo **sin** usar la palabra prohibida en la UI). El **valor** asociado siempre se expresa en **Créditos**. *(Si el titular del producto prefiere conservar la etiqueta literal del canal de efectivo, deberá evaluar el riesgo de interpretación; por defecto se usa «Pago presencial».)*

### 14.2 Tipo de torneo: Competitivo vs Recreativo

Propiedad del torneo `Quinielas.tipo ∈ {competitivo, recreativo}`, elegida por el **Comisionado** al **crear** el torneo (en `<ModalCrearQuiniela>`, §4.8) junto con el **formato** de puntuación (§5):

- **Competitivo** — gestiona **Créditos**: define un **costo de inscripción** (`costo_inscripcion_creditos`), consolida una **Bolsa Única** (§14.4) y reparte premios con la **calculadora de distribución** (§14.5), incluido el **Premio al Campeón** (§14.6). Habilita el **panel del Tesorero** y las vistas de premios.
- **Recreativo** — **gratuito y sin recompensas**: la app **oculta por completo** toda UI/lógica económica (sin costo, sin Bolsa, sin distribución, sin premios). Solo juego, ranking y avisos.

> **Inmutabilidad.** El `tipo` se fija **al crear** el torneo y **no** cambia después. El `costo_inscripcion_creditos` y la **distribución** se congelan según §14.4–§14.5 (alineado con el congelamiento de reglas de §5.5).

### 14.3 Modelo de datos económico (pestañas dedicadas de Google Sheets)

Para no dispersar la lógica, **todas** las estructuras de créditos/premios se definen **aquí** (en §6 solo se enumeran como puntero):

- **`Quinielas` (campos económicos, además de los de §6):** `tipo` (`competitivo`|`recreativo`), `costo_inscripcion_creditos` (entero ≥ 0; **0** o nulo en `recreativo`), `bolsa_congelada` (bool), `distribucion_congelada` (bool).
- **`Pagos`** — `id`, `id_quiniela`, `id_usuario`, `monto_creditos` (normalmente = `costo_inscripcion_creditos`), `metodo` (`transferencia`|`deposito`|`presencial`), `estatus` (`preinscrito`|`inscrito`), `confirmado_por` (`id_usuario` del Tesorero que confirma), `confirmado_en`, `nota`. Clave lógica única (`id_quiniela` + `id_usuario`).
- **`Bolsa`** (derivable/cacheable) — `id_quiniela`, `total_creditos` (**Σ** `monto_creditos` de los `Pagos` con `estatus = inscrito` de ese torneo), `num_inscritos`, `actualizada_en`.
- **`DistribucionPremios`** — `id_quiniela`, `concepto` (`fases_regulares`|`eliminacion_directa`|`campeon`|`retencion_admin`), `modo` (`porcentaje`|`fijo`), `valor` (% o Créditos según `modo`), `creditos_resultantes` (materializado al cuadrar/congelar), `congelado` (bool). **Una fila por concepto.**
- **`Premios`** (asignaciones resultantes) — `id`, `id_quiniela`, `id_usuario`, `concepto` (mismo dominio que arriba), `creditos`, `criterio` (texto auditable: p. ej. «posición regular #3 = campeón real #3»), `asignado_en`.

### 14.4 Recaudación externa y panel del **Tesorero** (Criterio: pagos)

- **Sin pasarelas de terceros.** El cobro **no** se integra con **Stripe**, **PayPal** ni ninguna pasarela automatizada. La recaudación es **externa a la plataforma**.
- **Pago directo al Tesorero.** Los usuarios pagan su inscripción **directamente al Tesorero** asignado por canales tradicionales: **transferencia bancaria**, **depósito** o **pago presencial** (§14.1). La app **no** mueve Créditos reales: solo **registra** el estado del cobro.
- **Flujo de inscripción (Competitivo):**
  1. Al **unirse** a un torneo Competitivo (`unirseQuiniela`, §7), el sistema crea un `Pago` con `estatus = preinscrito` y `monto_creditos = costo_inscripcion_creditos`.
  2. El usuario realiza el pago **offline** al Tesorero.
  3. El **Tesorero** lo confirma **manualmente** en su panel: `preinscrito → inscrito`, registrando `metodo` y `nota`.
  4. La **Bolsa Única** se **recalcula automáticamente** (Σ de Créditos de los `inscrito`).
- **`<PanelTesoreria>` (panel exclusivo del rol Tesorero; Comisionado por absorción, §8):**
  - Lista de **preinscritos** e **inscritos** del torneo, con su `monto_creditos` y `metodo`.
  - Acción para **cambiar el estatus** de un usuario a **«Inscrito»** (y, si aplica, revertir).
  - Indicador de la **Bolsa Única** del torneo, **en Créditos**, actualizada a partir de las confirmaciones.
  - **Autorización server-side por torneo** (solo `tesorero`/`comisionado`); ocultar el panel en UI es solo cosmético (§13 A01).

### 14.5 Calculadora de distribución de la **Bolsa Única** (Criterio: distribución)

Herramienta interna **`<CalculadoraDistribucion>`** dentro del **panel de administración** (rol `tesorero`/`comisionado`) para definir el **destino** de la **Bolsa Única** acumulada.

- **Cuatro conceptos obligatorios** a cubrir (`DistribucionPremios.concepto`):
  1. **`fases_regulares`** — premios por **jornadas o etapas** de la fase regular.
  2. **`eliminacion_directa`** — premios de **cada** fase de eliminación directa (liguillas/playoffs: octavos, cuartos, semifinal, final).
  3. **`campeon`** — **Premio al Campeón** (mecánica especial, §14.6).
  4. **`retencion_admin`** — **retención** por **gastos administrativos u organizativos** de la plataforma.
- **Modo de reparto:** por **porcentaje** (dinámico) **o** por **valor fijo** (en Créditos). Se puede combinar por concepto.
- **Validación de cuadre (server-side):** la suma debe **cuadrar exactamente** con la Bolsa: **Σ porcentajes = 100 %** y/o **Σ fijos + Σ(% · Bolsa) = `Bolsa.total_creditos`**, **sin exceso ni faltante**. La UI muestra en **vivo** los `creditos_resultantes` por concepto y **bloquea** la confirmación si no cuadra.
- **Congelamiento:** al confirmar, `DistribucionPremios.congelado = true` y `Quinielas.distribucion_congelada = true`; queda **inmutable** y registrado en `RegistroAuditoria` (§13).

### 14.6 **Premio al Campeón** — mecánica de **posición cruzada** (azar + rendimiento)

> **No** se otorga al jugador con **más puntos** en la plataforma. Es una mecánica singular que **cruza** el resultado del **torneo real** con la **tabla de la quiniela**.

**Definición:**
1. Al **finalizar la competencia real**, el sistema obtiene del **proveedor de API** (§6) la **posición final en la fase regular** (de la tabla de la **liga real**) del **equipo que se coronó campeón** del torneo físico. Llamemos a esa posición **`P`**.
2. El sistema busca, en la **tabla general de posiciones de la quiniela al término de su propia fase regular** (snapshot **congelado** de `Posiciones` de la etapa regular, §6), al **usuario que ocupa esa misma posición numérica `P`**.
3. Le **asigna automáticamente** el **Premio al Campeón** (`Premios.concepto = campeon`), por los Créditos definidos en `DistribucionPremios`.

**Ejemplo de programación:** si el equipo campeón del torneo real **finalizó la fase regular en el 3.er lugar** de la tabla de la liga, el **Premio al Campeón** de la app se otorga **exclusivamente** al **jugador que concluyó la fase regular de la quiniela en el 3.er puesto** de la tabla de usuarios.

**Reglas e invariantes (server-side, `resolverPremioCampeon(idQuiniela)`):**
- **Solo server-side**, **idempotente** y **auditable**; se dispara cuando el proveedor de API marca el **torneo real como concluido** (campeón definido). El cliente nunca lo resuelve ni lo edita.
- Requiere un **snapshot congelado** de las `Posiciones` de la quiniela **al cierre de su fase regular** (la posición `P` se busca sobre esa tabla, no sobre la tabla general final).
- **Solo cuentan jugadores** del ranking (los administradores en **solo-gestión** no aparecen, §8); `P` indexa la tabla de jugadores.
- **Empate en la posición `P` de la quiniela:** aplicar un **desempate determinista y documentado** (criterio sugerido: mayor número de **marcadores exactos** acumulados en la fase regular; si persiste, **reparto equitativo** del Premio al Campeón entre los empatados). Definir **una** regla y registrarla en `Premios.criterio`.
- **`P` fuera de rango** (la quiniela tiene **menos jugadores** que la posición `P`): aplicar el **fallback documentado** (sugerido: otorgar al **último lugar** de la quiniela, o **reasignar** ese monto a `retencion_admin`). Definir **una** regla y auditarla.

### 14.7 Capa de acceso a datos económica (endpoints — §7 solo referencia)

**Escritura (POST → Apps Script con validación y autorización por rol/torneo):**
- `crearQuiniela(..., tipo, costo_inscripcion_creditos?)` — extiende §7: fija `tipo`; si `competitivo`, captura el **costo de inscripción en Créditos**.
- `unirseQuiniela(codigo)` — en `competitivo`, crea además el `Pago` con `estatus = preinscrito` (§14.4).
- `confirmarInscripcion(idQuiniela, idUsuario, metodo, nota?)` — **solo Tesorero/Comisionado**; `preinscrito → inscrito`; **recalcula la Bolsa**.
- `revertirInscripcion(idQuiniela, idUsuario)` — **solo Tesorero/Comisionado**; `inscrito → preinscrito`; recalcula la Bolsa.
- `definirDistribucion(idQuiniela, reparto)` — **solo Tesorero/Comisionado**; valida **cuadre** y **congela** (§14.5).
- `resolverPremioCampeon(idQuiniela)` — **interno/server-side**; asigna el **Premio al Campeón** por posición cruzada (§14.6).

**Lectura (GET):**
- `obtenerBolsa(idQuiniela)` — `total_creditos` y `num_inscritos`.
- `obtenerDistribucion(idQuiniela)` — reparto por concepto (y `creditos_resultantes`).
- `obtenerPremios(idQuiniela)` — asignaciones de premios (para «Reglas» y `<PanelMiQuiniela>`).

### 14.8 UI económica (componentes — §4 solo referencia)

- **`<PanelTesoreria>`** (§14.4) y **`<CalculadoraDistribucion>`** (§14.5) dentro del **panel de administración**.
- En **«Reglas»** (§4.7) y **`<PanelMiQuiniela>`** (§4.4): mostrar **costo de inscripción**, **Bolsa Única**, **reparto de premios** y **premios asignados**, siempre **en Créditos**.
- En `tipo = recreativo`, **todos** estos elementos quedan **ocultos**.

### 14.9 Seguridad e integridad económica (§13 solo referencia)

- **Autorización:** confirmar pagos y definir la distribución son operaciones **exclusivas** de `tesorero`/`comisionado`, verificadas **server-side por torneo** (`Participaciones`, §8; **menor privilegio**, §13 A01).
- **Cliente nunca es fuente de verdad:** la **Bolsa**, los `creditos_resultantes` y las asignaciones de **premios** se calculan **server-side**; el cliente no fija montos.
- **Cuadre obligatorio:** la distribución se **bloquea** si no suma el total/100 %; tras congelar es **inmutable**.
- **Premio al Campeón verificado:** resuelto **server-side** desde la **tabla final del proveedor de API**, validando **origen/firma** de la respuesta antes de actuar (§13 A08/A10).
- **Auditoría:** todo cambio de `estatus` de pago, de **Bolsa** y de **distribución** queda en `RegistroAuditoria` (sin PII, §13 A09).
- **Sanitización:** `nota` y textos libres se sanitizan contra **formula/CSV injection** (`= + - @`, §13 A03).

### 14.10 Cumplimiento legal (recordatorio)

La terminología **«Créditos»** y el cobro **externo** (sin pasarelas) **reducen**, pero **no eliminan**, el riesgo de que el producto se interprete como apuestas/sorteos. Antes de operar torneos **Competitivos**, revisar la **legalidad local** (apuestas, sorteos, rifas, juego con premio); esto **excede lo técnico** y es responsabilidad del organizador.

### Checklist de §14 (criterio de aceptación)
- [ ] **Terminología:** en **toda** la UI el valor se denomina **«Créditos»**; **sin** «dinero/pesos/dólares/efectivo» ni símbolos de moneda para valor in-app; claves i18n en «Créditos».
- [ ] **Tipo de torneo:** **Competitivo**/**Recreativo** elegible al **crear**; **Recreativo** oculta **toda** UI y lógica económica.
- [ ] **Pago externo (sin Stripe/PayPal):** al unirse a un Competitivo se crea **preinscripción**; el **Tesorero** confirma **manualmente** → **«Inscrito»**.
- [ ] **Panel del Tesorero:** lista de **preinscritos/inscritos** y cambio de estatus; **Bolsa Única = Σ Créditos** de inscritos, **server-side**.
- [ ] **Calculadora de distribución:** **4 conceptos** (`fases_regulares`, `eliminacion_directa`, `campeon`, `retencion_admin`), por **porcentaje** o **fijo**, **cuadrando** el total; **congela** al confirmar.
- [ ] **Premio al Campeón:** **posición cruzada** (posición del **campeón real** en fase regular ↔ **jugador** en esa posición de la quiniela), resuelto **server-side** desde la **tabla final verificada**; **desempates** y **`P` fuera de rango** definidos y auditados.
- [ ] **Integridad:** autorización **Tesorero/Comisionado**; cálculos **server-side**; **inmutabilidad** tras congelar; **auditoría** de pagos/distribución.

---

## 15. Protección de propiedad intelectual (PI) y disuasión legal

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
Incluir en el `<head>` del `index.html` (junto a la CSP de §13 A05):

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
Configurado en `vite.config.js` (sección 11), como parte de la PI:
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

## 16. Orden de implementación sugerido

1. Scaffolding **en un único `index.html`** (Vite + `vite-plugin-singlefile`): `<style>` embebido con **tokens + temas oscuro/claro**, **marco móvil con radio de aspecto**, **i18n es/en como objetos JS embebidos**, `ProveedorTema`/`ProveedorIdioma` y un **enrutador en memoria**; layout con encabezado/BarraJornadas/nav inferior fijos. Incluir desde el inicio `<SelectorTema>` y `<SelectorIdioma>` con persistencia. **Configurar la PI (sección 15) desde el primer commit:** `LICENSE` propietario, `public/robots.txt` (`Disallow` + `noai`/`noindex`), meta-tags de `<head>` (`tdm-reservation: 1`, `robots noai,noindex`, copyright) y `vite.config.js` endurecido (sin sourcemaps, sin comentarios, `mangle`, salida de un solo archivo).
2. Pantalla **Jugar** con **datos de prueba** (jornada, partidos, entrada de pronóstico) hasta lograr paridad visual y funcional (sección 12), validando ambos temas y ambos idiomas.
3. Apps Script + estructura de Sheets (incluida `Participaciones`); lectura (`obtenerMisQuinielas`, `obtenerJornadaActiva`, `obtenerPartidosJornada`, `obtenerMisPronosticos`). **Aplicar desde aquí los controles de la sección 13** (validación, CORS, sanitización, **fecha límite server-side**).
4. Integración con el proveedor de API de resultados deportivos en vivo: sincronización periódica/webhook que actualiza `Partidos` (estado, marcador, horarios) — ver sección 6.
5. Auth (Google → alternativa correo) + verificación de teléfono por OTP + **roles por torneo**, con verificación de `id_token` server-side.
6. **Crear y unirse a torneos:** `crearQuiniela` (selección de **formato** —Casual/Incremental/Personalizado, §5—, captura de reglas por fase para `personalizado` **antes del lanzamiento**, y generación del **código único de 6 caracteres** server-side) y `unirseQuiniela(codigo)` con validación/rate limiting; selector de torneo activo y cambio de contexto.
7. Envío de pronósticos con bloqueo por fecha límite (**15 min antes del inicio de cada partido**); lectura del ranking (`obtenerTablaPosiciones`) y del **estado de jugadores** (`obtenerEstadoJugadores`, §4.4.1: Atrasado / En Riesgo / Al día) del torneo activo.
8. Calificación automática de jornadas: `calificarJornada` se dispara cuando el proveedor de API marca los partidos como `finalizado` (**cálculo de puntos con el algoritmo de la §5.4**: formato + fase, sobre reglas congeladas §5.5) + `RegistroAuditoria`. Panel **admin** (por torneo) para gestión de jornadas/participantes/avisos.
9. **Créditos y premios (§14, solo torneos `tipo=competitivo`):** `tipo` de torneo y **costo de inscripción en Créditos** en `crearQuiniela`; **panel del Tesorero** (`preinscrito`→`inscrito`) y consolidación de la **Bolsa Única**; **calculadora de distribución** (fases regulares / eliminación directa / **Campeón** / retención administrativa, por **porcentaje** o **fijo**, validando el total) con **congelamiento**; **Premio al Campeón** por **posición cruzada** (`resolverPremioCampeon`) desde la **tabla final** del proveedor de API; terminología **«Créditos»** en toda la UI y `RegistroAuditoria` de pagos/distribución.
10. EspacioPublicidad en iframe sandbox; CI/CD a GitHub Pages (`npm ci` + `npm audit`); estados de carga/vacío/cerrado/error; pulido del marco responsive (móvil + radio de aspecto en desktop).

---

### Definición de "hecho"
La web app **administra la quiniela**: el usuario autenticado (Google + teléfono verificado, o correo) puede **crear múltiples torneos privados y unirse a otros, exclusivamente por invitación, mediante un código único de 6 caracteres alfanuméricos**, y en el torneo activo captura pronósticos de la jornada abierta que se **bloquean automáticamente 15 minutos antes del inicio de cada partido** (`fecha_limite = inicio_partido − 15 min`, validado server-side); los resultados oficiales se **sincronizan automáticamente desde un proveedor de API de resultados deportivos en vivo** y disparan el **cálculo de puntos** según el **formato del torneo** (Casual / Incremental / Personalizado) y sus **reglas por fase congeladas al lanzamiento**, definidos de forma única y cerrada en la **sección 5**; el **ranking** de participantes se actualiza y se muestra —junto con el **estado de cada jugador** (**Atrasado**, **En Riesgo** o **Al día**, §4.4.1)—, todo respetando los **roles por torneo** (rol base **Jugador** + un **equipo administrativo** de Comisionado/Tesorero/Operador/Vocal, con **menor privilegio**; un único Comisionado que asigna/transfiere roles y expulsa jugadores, **sin abandono voluntario**; §8). En los torneos **Competitivos**, además, se gestionan **Créditos** según **§14** (costo de inscripción, **Bolsa Única** consolidada por el **Tesorero** a partir de pagos **externos** —sin pasarelas de terceros—, **calculadora de distribución** en fases regulares / eliminación directa / **Campeón** / retención administrativa, y **Premio al Campeón** por **posición cruzada** entre el torneo real y la quiniela), empleando **exclusivamente** la terminología «Créditos» en la UI. La interfaz es **exclusivamente móvil** (en desktop conserva el **radio de aspecto** en un marco centrado), soporta **tema oscuro/claro** (oscuro por defecto) e **idiomas español/inglés** (español por defecto), ambos conmutables y persistentes, conserva la estética de la captura (sección 12) y pasa el **checklist de seguridad OWASP** (sección 13), desplegada en GitHub Pages bajo HTTPS con CORS y CSP correctos, como un **único `index.html` autocontenido (SPA estricta de archivo único)** —con toda navegación resuelta **en memoria**, sin rutas ni archivos HTML adicionales— y con la **capa de protección de PI activa** (licencia restrictiva *Todos los derechos reservados*, `robots.txt` con `noai`/`noindex`, meta `tdm-reservation`, build sin source maps; sección 15).
