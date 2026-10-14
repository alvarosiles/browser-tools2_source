# IT Support Tools (browser-tools)

Panel web de soporte técnico para equipos **Windows y Linux**. La interfaz (React + Vite + Tailwind CSS) se publica como sitio estático, y todas las acciones reales sobre el sistema operativo las ejecuta un **agente local** en Node.js/Express que corre en la máquina del técnico y escucha solo en `127.0.0.1`.

- Repositorio: `https://github.com/alvarosiles/browser-tools`
- Sitio publicado (GitHub Pages, según `vite.config.js`, `package.json` y los scripts de instalación): `https://alvarosiles.github.io/browser-tools/`

---

## 1. Descripción general

El navegador no puede ejecutar comandos del sistema por sí mismo. Este proyecto separa el problema en dos piezas:

1. **Panel web (frontend)**: una SPA que agrupa las tareas de soporte en tarjetas (navegadores, Windows, impresoras, red, información del equipo, carpetas, reparaciones).
2. **Agente local (`local-agent/`)**: un servidor HTTP local que recibe las peticiones del panel y ejecuta PowerShell, `cmd`, `netsh`, `robocopy`, CUPS, `nmcli`, `systemctl`, etc.

El agente se distribuye como **binario autocontenido (Node SEA)**, de modo que el técnico no necesita tener Node.js instalado: se instala con un comando, se registra para arrancar con el sistema y se autoactualiza.

## 2. Objetivo y problema que resuelve

Reunir en una sola pantalla las tareas repetitivas de soporte técnico de primer nivel, que normalmente requieren abrir muchas herramientas distintas:

- Borrar datos de navegación (por tipo, por navegador o por dominio) y resetear perfiles.
- Respaldar perfiles, marcadores y acceso a contraseñas de los navegadores antes de limpiarlos.
- Gestionar impresoras y la cola de impresión.
- Consultar el estado de red y los datos del equipo.
- Lanzar reparaciones habituales de Windows (SFC, DISM, CHKDSK, DNS, Winsock).

## 3. Características principales

- 10 navegadores soportados: Chrome, Edge, Firefox, Brave, Opera, GNOME Web (Epiphany), Chromium, Vivaldi, LibreWolf y Zen Browser, con detección de los que están instalados.
- Borrado selectivo de 13 tipos de datos (`cookies`, `cache`, `localStorage`, `sessionStorage`, `indexedDB`, `serviceWorkers`, `history`, `webSQL`, `cacheStorage`, `webAppManifest`, `autofillForms`, `permissions`, `push`) en todos los perfiles del navegador.
- Borrado de datos de **un solo dominio** editando las bases SQLite de historial y cookies con `better-sqlite3`.
- Operaciones largas (borrado múltiple, reset, respaldos, reparaciones) ejecutadas como **jobs en segundo plano** con consulta de progreso, para que no se corten si se cierra el navegador que aloja el panel.
- Soporte multiplataforma Windows/Linux en el agente (incluye rutas de perfiles instalados vía snap).
- Interfaz bilingüe (español/inglés) y tema claro/oscuro, persistidos en `localStorage`.
- Instalación del agente con un comando, arranque automático con el sistema y autoactualización desde GitHub Pages.
- Pausa/reanudación del agente desde el panel (botón Play/Stop) mediante un puerto de control separado.

## 4. Arquitectura

```
┌──────────────────────────────┐        HTTP (fetch, JSON)        ┌──────────────────────────────────┐
│  Panel web (React SPA)       │ ───────────────────────────────▶ │  Agente local (Node + Express)   │
│  GitHub Pages / Vite dev     │   localhost:5177  (worker)       │  127.0.0.1:5177  acciones        │
│  src/lib/localAgent.js       │   localhost:5178  (control)      │  127.0.0.1:5178  start/stop,     │
└──────────────────────────────┘                                  │                  versión, update │
                                                                  └───────────────┬──────────────────┘
                                                                                  │ child_process
                                                                                  ▼
                                                     PowerShell / CIM, cmd, netsh, robocopy, taskkill (Windows)
                                                     lpstat/lp (CUPS), nmcli, systemctl, xdg-open (Linux)
                                                     better-sqlite3 sobre History / places.sqlite / cookies
```

- **Frontend**: SPA sin router. `App.jsx` monta `Header`, `Dashboard`, `Footer` y `Toast`. Toda la comunicación con el agente está centralizada en [src/lib/localAgent.js](src/lib/localAgent.js).
- **Agente**: dos aplicaciones Express en el mismo proceso:
  - **Worker** (`PORT`, por defecto `5177`): expone las acciones. Se puede apagar y volver a encender.
  - **Control** (`CONTROL_PORT`, por defecto `5178`): siempre activo; permite `start`/`stop`, consultar versión, forzar actualización y desinstalar.
- **Jobs**: los procesos largos devuelven un `jobId` (`crypto.randomUUID()`) y se guardan en `Map` en memoria; el frontend consulta su estado periódicamente.
- **Distribución**: `esbuild` empaqueta `server.js` a CommonJS → `node --experimental-sea-config` genera el blob → `postject` lo inyecta en una copia del ejecutable de Node.

## 5. Estructura del proyecto

```
browser-tools/
├── index.html                 # Punto de entrada HTML de Vite
├── vite.config.js             # React + Tailwind; base '/browser-tools/' si GITHUB_PAGES
├── package.json               # Scripts y dependencias del panel web
├── dev.sh                     # Arranque en desarrollo (nvm use + npm install si falta + web:run)
├── .nvmrc                     # Node 22
├── .oxlintrc.json             # Reglas de oxlint (plugins react y oxc)
├── src/
│   ├── main.jsx               # Monta <App/> dentro de ThemeProvider y LanguageProvider
│   ├── App.jsx                # Layout: Header, Dashboard, Footer, Toast
│   ├── index.css              # Tailwind v4
│   ├── components/
│   │   ├── Header.jsx             # Título + ServiceToggle
│   │   ├── ServiceToggle.jsx      # Play/Stop del agente (puerto de control)
│   │   ├── LocalAgentStatus.jsx   # Aviso si el agente no responde + comando de instalación
│   │   ├── Dashboard.jsx          # Grilla de tarjetas
│   │   ├── Card.jsx               # Contenedor visual reutilizable
│   │   ├── SettingsPanel.jsx      # Tema e idioma
│   │   ├── BrowserManager.jsx     # Borrado, borrado por dominio, reset y respaldo de navegadores
│   │   ├── WindowsTools.jsx       # Reloj y accesos a herramientas del sistema
│   │   ├── PrinterTools.jsx       # Impresoras, cola y spooler
│   │   ├── NetworkStatus.jsx      # Interfaces, Wi-Fi activo y redes guardadas
│   │   ├── SystemInfo.jsx         # Datos del equipo
│   │   ├── QuickFolders.jsx       # Carpetas rápidas y apps de acceso remoto
│   │   ├── SystemRepair.jsx       # SFC, DISM, CHKDSK, DNS, Winsock
│   │   ├── Footer.jsx / Toast.jsx
│   ├── hooks/useToast.js
│   └── lib/
│       ├── localAgent.js      # Único punto de integración HTTP con el agente
│       ├── i18n.jsx           # Contexto de idioma (es/en)
│       ├── translations.js    # Textos de la interfaz
│       └── theme.jsx          # Contexto de tema claro/oscuro
├── local-agent/
│   ├── server.js              # Express (worker + control), jobs, autoinstalación, desinstalación
│   ├── commands.js            # Implementación de todas las acciones (Windows y Linux)
│   ├── updater.js             # Autoactualización vía version.json
│   ├── build-exe.js           # Empaquetado Node SEA
│   ├── install.ps1 / install.sh / install-linux.sh / Iniciar-Servicio.bat
│   └── package.json
├── scripts/
│   └── build-agent-exe.js     # Genera el binario y lo copia a public/downloads/ con version.json
├── public/
│   ├── admin/                 # Página estática con los comandos de instalación
│   ├── agent-status/          # Página estática: estado del agente local
│   ├── agent-update/          # Página estática: forzar actualización
│   ├── agent-uninstall/       # Página estática: desinstalar el agente
│   └── favicon.svg, icons.svg
├── alvavroindex.html, siles/browservault.html   # HTML estáticos independientes (no referenciados por la app)
└── *.md (INFO.md, mac.md, z.md, ...)            # Notas de trabajo
```

`public/downloads/` (binarios, scripts de instalación y `version.json`) se genera con `scripts/build-agent-exe.js` y está en `.gitignore`.

## 6. Tecnologías y versiones

Versiones declaradas en `package.json` y resueltas en `package-lock.json`:

| Capa | Tecnología | Declarada | Resuelta (lock) |
|---|---|---|---|
| Runtime | Node.js | `.nvmrc`: `22` | — |
| Lenguaje | JavaScript ESM + JSX | `"type": "module"` | — |
| UI | React / React DOM | `^19.2.8` | 19.2.8 |
| Bundler | Vite | `^8.2.0` | 8.2.0 |
| Plugin | @vitejs/plugin-react | `^6.0.4` | 6.0.5 |
| Estilos | Tailwind CSS + @tailwindcss/vite | `^4.3.3` | 4.3.3 |
| Iconos | lucide-react | `^1.28.0` | 1.28.0 |
| Linter | oxlint | `^1.75.0` | 1.77.0 |
| Scripts | concurrently / cross-env / gh-pages | `^10.0.4` / `^10.1.0` / `^6.3.0` | 10.0.4 / 10.1.0 / 6.3.0 |
| Agente | express | `^4.19.2` | 4.22.2 |
| Agente | cors | `^2.8.5` | 2.8.6 |
| Agente | better-sqlite3 | `^12.4.1` | 12.11.1 |
| Build agente | esbuild / postject | `^0.28.2` / `^1.0.0-alpha.6` | 0.28.2 / 1.0.0-alpha.6 |

También están declarados `postcss` y `autoprefixer`, pero no hay archivo de configuración de PostCSS: Tailwind v4 se integra a través de `@tailwindcss/vite`. `@types/react` y `@types/react-dom` están como devDependencies aunque el proyecto no tiene archivos TypeScript.

Requisito de Node: Vite 8, `@vitejs/plugin-react` 6 y oxlint exigen `^20.19.0 || >=22.12.0`; `concurrently` 10 exige `>=22`. Por eso se recomienda Node 22.

## 7. Módulos y funcionalidades

| Tarjeta (componente) | Funcionalidad verificada en el código |
|---|---|
| **Personalización** (`SettingsPanel`) | Tema claro/oscuro e idioma español/inglés. |
| **Borrar y respaldar navegadores** (`BrowserManager`) | Matriz navegador × tipo de dato para borrar (7 tipos básicos + 6 en "Detallado"); borrado de datos de un dominio en todos los navegadores instalados; reset completo de perfil; respaldo de perfil completo, marcadores y apertura del gestor de contraseñas; apertura de la carpeta de respaldos. Pide confirmación (`window.confirm`) antes de procesar. |
| **Control de Windows** (`WindowsTools`) | Reloj en tiempo real y accesos a Panel de Control, Configuración, Administrador de tareas, CMD como administrador (UAC), PowerShell, Servicios y Administrador de dispositivos (equivalentes GNOME en Linux). |
| **Control de impresión** (`PrinterTools`) | Listado de impresoras, impresora predeterminada, página de prueba (incluye impresoras "a archivo" como Microsoft Print to PDF sin diálogo), propiedades, vaciar cola, eliminar trabajos atascados y reiniciar el spooler. |
| **Estado de red** (`NetworkStatus`) | Interfaces de red (marcando virtuales), Wi-Fi activo (SSID, señal, banda) y redes Wi-Fi guardadas; en Windows incluye la contraseña, oculta por defecto con botón para mostrarla. |
| **Información del PC** (`SystemInfo`) | Nombre del equipo, usuario, SO, RAM, CPU, disco, IP local, IP pública (consulta a `api.ipify.org`), dominio/grupo de trabajo y número de serie. |
| **Accesos rápidos** (`QuickFolders`) | Descargas, Temp, AppData, Inicio; apertura de TeamViewer y AnyDesk si están instalados. |
| **Reparaciones automáticas** (`SystemRepair`) | SFC, DISM RestoreHealth, CHKDSK `/scan` (solo lectura), Flush DNS, Reset Winsock y "todo en un clic" como job; detecta si el agente corre con privilegios de administrador. En Linux usa equivalentes (`journalctl`, `apt-get check`, `df`, `resolvectl`, reinicio de NetworkManager). |
| **Estado del agente** (`LocalAgentStatus`, `ServiceToggle`) | Comprueba el agente cada pocos segundos; si no responde muestra el comando de instalación según el SO; Play/Stop del servicio. |

No hay autenticación de usuarios, roles ni permisos en la aplicación.

## 8. Base de datos

El proyecto **no tiene base de datos propia**, ni modelos, esquemas o migraciones. `better-sqlite3` se usa únicamente para **editar las bases SQLite de los navegadores** al borrar los datos de un dominio:

| Navegador | Archivo | Tablas afectadas |
|---|---|---|
| Chromium (Chrome, Edge, Brave, Opera, Chromium, Vivaldi) | `Cookies`, `History` | `cookies` (`host_key`), `urls`, `visits` |
| Firefox, LibreWolf, Zen | `cookies.sqlite`, `places.sqlite` | `moz_cookies` (`host`), `moz_places`, `moz_historyvisits` |
| GNOME Web | `cookies.sqlite`, `ephy-history.db` | `moz_cookies`, `urls`, `visits`, `hosts` |

El estado de los jobs vive en memoria (`Map`) y se pierde al reiniciar el agente. Las preferencias del panel (tema, idioma) se guardan en `localStorage`.

## 9. Backend y API (agente local)

Todas las respuestas son JSON con la forma `{ ok: true, ... }` o `{ ok: false, error }`. Las acciones `POST` registradas con `handle()` devuelven `{ ok, action, result }`.

### Worker — `http://127.0.0.1:5177`

| Método | Ruta | Body |
|---|---|---|
| GET | `/health`, `/version` | — |
| GET | `/installed-browsers`, `/printers`, `/system-info`, `/is-admin`, `/network-status` | — |
| POST | `/clear-browser-data` | `{ browserId, types[] }` |
| POST | `/reset-browser-profile` | `{ browserId }` |
| POST | `/clear-domain-data` | `{ domain }` |
| POST | `/backup-browser-profile`, `/backup-browser-bookmarks`, `/open-password-manager` | `{ browserId }` |
| POST | `/open-backup-folder` | `{ date? }` |
| POST | `/open-control-panel`, `/open-windows-settings`, `/open-task-manager`, `/open-cmd-admin`, `/open-powershell`, `/open-services`, `/open-device-manager` | — |
| POST | `/open-printer-maintenance` | `{ printerName? }` |
| POST | `/print-test-page`, `/set-default-printer`, `/clear-print-queue`, `/remove-stuck-jobs` | `{ printerName }` |
| POST | `/restart-spooler` | — |
| POST | `/open-quick-folder` | `{ folderKey: downloads \| temp \| appdata \| startup }` |
| POST | `/open-remote-app` | `{ appId: teamviewer \| anydesk }` |
| POST | `/sfc-scan`, `/dism-restore-health`, `/chkdsk-scan`, `/flush-dns`, `/reset-winsock` | — |

### Jobs en segundo plano

| Inicio (POST) | Body | Estado (GET) |
|---|---|---|
| `/backup-all` | — | `/backup-status/:jobId` |
| `/backup-selected` | `{ items }` | `/backup-selected-status/:jobId` |
| `/clear-browsers-data` | `{ items }` | `/clear-browsers-data-status/:jobId` |
| `/reset-browser-profiles` | `{ browserIds }` | `/reset-browser-profiles-status/:jobId` |
| `/run-all-repairs` | — | `/repair-status/:jobId` |

### Control — `http://127.0.0.1:5178`

| Método | Ruta | Acción |
|---|---|---|
| GET | `/status` | `{ running }` del worker |
| POST | `/start`, `/stop` | Enciende / apaga el worker |
| GET | `/version` | Versión del agente y si corre como binario SEA |
| POST | `/update-now` | Fuerza la búsqueda de actualización (solo en binario SEA) |
| POST | `/uninstall` | Quita el arranque automático y borra la carpeta de instalación |

### Validaciones existentes

- `clearBrowserData` filtra los tipos contra la lista `BROWSER_DATA_TYPES` y exige al menos uno.
- `normalizeDomain` exige un dominio y lo reduce al `hostname` con `new URL()`.
- `openQuickFolder` y `openRemoteApp` solo aceptan claves de listas cerradas.
- En PowerShell, los nombres de impresora se escapan y los scripts se envían con `-EncodedCommand`.
- Antes de borrar, el navegador se cierra con reintentos hasta confirmar que no se relanza.

## 10. Configuración e instalación

Requisitos:

- Node.js 22 (`.nvmrc`). Con `nvm`: `nvm use` en la raíz.
- npm.
- Windows o Linux para el agente. En Linux, CUPS (`cups`, `cups-client`) para impresoras y NetworkManager (`nmcli`) para Wi-Fi.
- `bash` para los scripts `dev`/`web` (`dev.sh`); en Windows, por ejemplo, Git Bash.

```bash
npm install
npm install --prefix local-agent
```

## 11. Variables de entorno

| Variable | Dónde | Uso | Por defecto |
|---|---|---|---|
| `PORT` | `local-agent/server.js` | Puerto del worker | `5177` |
| `CONTROL_PORT` | `local-agent/server.js` | Puerto de control | `5178` |
| `GITHUB_PAGES` | `vite.config.js` | Si existe, `base = '/browser-tools/'` | sin definir (`/`) |

El frontend tiene fijadas las URLs `http://localhost:5177` y `http://localhost:5178` en `src/lib/localAgent.js`, así que si se cambian los puertos del agente también hay que cambiar ese archivo. No hay archivos `.env` ni secretos en el repositorio.

## 12. Ejecución en desarrollo

```bash
# Agente + panel juntos (usa dev.sh: nvm use, npm install si faltan módulos, concurrently)
npm run dev

# Equivalente sin dev.sh
npm run web:run

# Por separado
npm start --prefix local-agent   # agente en 127.0.0.1:5177 / 5178
npx vite                         # panel en http://localhost:5173
```

Ejecutado con `npm start`, el agente corre en la terminal actual y no se autoinstala ni se autoactualiza: eso solo ocurre cuando corre como binario SEA.

## 13. Compilación y despliegue

```bash
npm run build      # build de producción en dist/
npm run preview    # sirve dist/ localmente

npm run build:exe --prefix local-agent   # solo el binario del agente (local-agent/dist/)

npm run deploy     # build del agente + build web con GITHUB_PAGES=true + publicación con gh-pages
```

Detalles de `npm run deploy`:

1. `scripts/build-agent-exe.js` comprueba que `local-agent/package.json` y `AGENT_VERSION` en `updater.js` coincidan, genera el binario y copia a `public/downloads/` el binario, `install.ps1`, `install.sh`, `install-linux.sh`, el código fuente del agente y `version.json`.
2. Node SEA no hace compilación cruzada: el binario de Windows solo se genera en Windows y el de Linux en Linux. Como `gh-pages` reemplaza todo el árbol publicado, hay que tener ambos binarios en `public/downloads/` antes de publicar.
3. Vite compila con `base: '/browser-tools/'` y `gh-pages -d dist` publica en GitHub Pages.

Instalación del agente para técnicos (sin Node.js):

```powershell
# Windows
irm https://alvarosiles.github.io/browser-tools/downloads/install.ps1 | iex
```

```bash
# Linux (systemd --user)
curl -fsSL https://alvarosiles.github.io/browser-tools/downloads/install.sh | bash
```

En Windows el binario se copia a `%LOCALAPPDATA%\BrowserToolsAgent\` y se registra en `HKCU\...\Run` mediante un `.vbs` que lo lanza sin ventana. En Linux se instala en `~/.local/share/browser-tools-agent/` con una unidad `systemd --user`. El binario instalado consulta `version.json` cada 6 horas y se reemplaza a sí mismo si hay una versión nueva.

## 14. Seguridad y consideraciones importantes

- **Alcance local**: ambos servidores escuchan solo en `127.0.0.1`, así que no son accesibles desde otras máquinas de la red.
- **Sin autenticación y con CORS abierto**: el agente usa `cors()` sin restricción de origen y no requiere token. Cualquier página abierta en el mismo equipo podría llamar a la API local, incluidas acciones destructivas (borrar datos, desinstalar) y lecturas sensibles (`/network-status` devuelve las contraseñas Wi-Fi guardadas en texto plano en Windows). Se recomienda restringir los orígenes permitidos y añadir un token compartido.
- **Construcción de comandos**: algunos parámetros recibidos por HTTP se interpolan en comandos de shell (por ejemplo, `printerName` en `openPrinterMaintenance` y `date` en `openBackupFolder`). Conviene validarlos con listas blancas o pasarlos como argumentos y no dentro de cadenas de comando.
- **Acciones destructivas**: borrar datos o resetear perfiles cierra el navegador (todas sus ventanas) y borrar cookies cierra las sesiones iniciadas. El panel pide confirmación antes de procesar.
- **Privilegios**: SFC, DISM, CHKDSK y Reset Winsock requieren administrador; el panel muestra si el agente está elevado. CMD como administrador se abre con UAC.
- **Datos personales**: "Información del PC" expone usuario, IPs y número de serie, y consulta la IP pública a `api.ipify.org`.
- **Cadena de actualización**: el agente descarga y ejecuta binarios desde GitHub Pages sin verificar firma ni hash. La integridad depende del control de esa cuenta.
- **Respaldos**: se guardan en `C:\ITSupport\Backups\<fecha>` (Windows) o `~/ITSupport/Backups/<fecha>` (Linux), y pueden incluir datos sensibles del perfil del navegador.

## 15. Limitaciones identificadas

- No hay pruebas automatizadas ni pipeline de CI en el repositorio.
- Los jobs se guardan en memoria y nunca se eliminan del `Map` mientras el agente está activo.
- Las URLs del agente están fijadas en el frontend; no se configuran por entorno.
- Sin `node_modules` y con un Node anterior a 20.19 el proyecto no arranca. `dev.sh` necesita `bash` y `nvm`.
- El binario SEA debe compilarse en cada sistema operativo; el despliegue publica lo que haya en `public/downloads/` en ese momento.
- Parte del análisis de salidas de `netsh` depende del idioma del sistema (se reconocen solo etiquetas en español e inglés).
- En Linux varias reparaciones son equivalentes aproximados (por ejemplo, `df -h /` en lugar de CHKDSK) y algunas requieren `sudo` sin contraseña.
- `BrowserManager.jsx` arranca con un dominio de ejemplo ya cargado en el campo "Borrar datos de un dominio".
- El repositorio contiene archivos HTML independientes (`alvavroindex.html`, `siles/browservault.html`) y notas `.md` de trabajo que no forman parte de la aplicación.
