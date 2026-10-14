# IT Support Tools

## Descripción breve (tarjeta)

Panel web de soporte técnico para Windows y Linux que, a través de un agente local en Node.js, limpia y respalda navegadores, gestiona impresoras, diagnostica la red y ejecuta reparaciones del sistema desde una sola interfaz.

**Etiquetas:** React 19 · Vite 8 · Tailwind CSS 4 · Node.js · Express · SQLite · PowerShell · Node SEA · GitHub Pages

## Descripción completa

IT Support Tools es una herramienta para técnicos de soporte que reúne en un solo panel las tareas que habitualmente obligan a abrir muchas utilidades del sistema por separado.

La solución tiene dos piezas:

- **Panel web** en React, publicado como sitio estático en GitHub Pages, bilingüe (español/inglés) y con tema claro/oscuro.
- **Agente local** en Node.js/Express, que corre en la máquina del técnico, escucha solo en `127.0.0.1` y ejecuta las acciones reales con PowerShell/CIM, `cmd`, `netsh` y `robocopy` en Windows, o con CUPS, `nmcli`, `systemctl` y `xdg-open` en Linux.

El agente se distribuye como un ejecutable autocontenido (Node Single Executable Application). Se instala con un solo comando, arranca con el sistema sin mostrar ventanas y se actualiza solo.

## Problema que resuelve

El soporte de primer nivel repite las mismas operaciones a diario: limpiar cachés y cookies, salir de una sesión de un sitio concreto, respaldar un perfil antes de resetearlo, destrabar la cola de impresión o correr SFC/DISM. Hacerlo a mano implica conocer rutas de perfiles de cada navegador, comandos de consola y paneles del sistema distintos. Además, un navegador no puede ejecutar comandos del sistema por razones de seguridad.

El proyecto resuelve ambas cosas: centraliza las tareas en una interfaz única y delega la ejecución en un servicio local controlado.

## Objetivo

Reducir el tiempo y los errores en tareas repetitivas de soporte técnico, ofreciendo acciones de un clic que sean seguras de repetir sobre cualquier equipo Windows o Linux, sin exigir que el técnico instale Node.js ni deje terminales abiertas.

## Público objetivo

- Técnicos de soporte y mesa de ayuda.
- Personal de TI que mantiene equipos de usuarios finales en Windows y Linux.

## Mi participación técnica

- **Confirmado por el historial de Git:** los 54 commits del repositorio (del 06/08/2026 al 25/08/2026) están firmados por Alvaro Siles Estrada / `alvarosiles`, y el sitio se publica bajo la cuenta `alvarosiles` en GitHub Pages.
- **Pendiente de confirmar:** el reparto concreto de responsabilidades (diseño de la interfaz, arquitectura del agente, empaquetado, despliegue) y si hubo otros colaboradores o asistencia de herramientas fuera del historial de Git.

## Funcionalidades destacadas

- **Gestión de navegadores (10 navegadores):** Chrome, Edge, Firefox, Brave, Opera, GNOME Web, Chromium, Vivaldi, LibreWolf y Zen. Detecta cuáles están instalados.
- **Borrado selectivo:** 13 tipos de datos (cookies, caché, localStorage, IndexedDB, service workers, historial, permisos, autocompletado, etc.) en todos los perfiles del navegador, con un solo cierre del navegador por operación.
- **Borrado por dominio:** elimina historial, cookies y almacenamiento de un único sitio editando directamente las bases SQLite de cada navegador, sin tocar el resto.
- **Respaldo y reset de perfiles:** copia de perfiles completos y marcadores en carpetas por fecha, acceso al gestor de contraseñas y reset completo del perfil.
- **Impresoras:** listado, impresora predeterminada, página de prueba (incluida "Microsoft Print to PDF" sin diálogo interactivo), vaciar cola, eliminar trabajos atascados y reiniciar el spooler.
- **Red e información del equipo:** interfaces, Wi-Fi activo, redes guardadas, IP local y pública, CPU, RAM, disco, dominio y número de serie.
- **Reparaciones automáticas:** SFC, DISM, CHKDSK de solo lectura, Flush DNS y Reset Winsock, por separado o en secuencia con progreso.
- **Accesos rápidos:** herramientas del sistema, carpetas frecuentes, TeamViewer y AnyDesk.
- **Control del servicio:** detección del agente, comando de instalación según el sistema operativo y botón Play/Stop.

## Tecnologías utilizadas

| Categoría | Tecnologías |
|---|---|
| Frontend | React 19.2, Vite 8.2, Tailwind CSS 4.3, lucide-react |
| Backend local | Node.js 22, Express 4, cors, better-sqlite3 12 |
| Automatización del sistema | PowerShell / CIM (WMI), cmd, netsh, robocopy, taskkill, rundll32 (Windows); CUPS, nmcli, systemctl, xdg-open (Linux) |
| Empaquetado y distribución | esbuild, Node SEA (`--experimental-sea-config`), postject |
| Despliegue | GitHub Pages, gh-pages, cross-env |
| Calidad y herramientas | oxlint, concurrently, nvm (`.nvmrc` = 22) |
| Lenguaje | JavaScript (ESM) y JSX |

## Arquitectura y características técnicas

- **Arquitectura cliente / agente local:** el panel es una SPA estática que se comunica por HTTP con el agente local. Toda la integración está concentrada en un único módulo (`src/lib/localAgent.js`).
- **Doble servidor en el agente:** un *worker* (puerto 5177) con las acciones y un *servidor de control* (puerto 5178) siempre activo, que permite pausar y reanudar el servicio, consultar la versión, forzar actualizaciones y desinstalar sin cerrar la aplicación.
- **Jobs asíncronos con sondeo:** las operaciones largas (respaldos, borrados múltiples, reparaciones) devuelven un `jobId` y el panel consulta su progreso. Así la operación termina aunque se cierre el navegador que aloja el panel, cosa que ocurre cuando se limpia ese mismo navegador.
- **Robustez frente al sistema operativo:** cierre de navegadores con reintentos hasta confirmar que no se relanzan, scripts de PowerShell enviados con `-EncodedCommand` para evitar problemas de comillas, `chcp 65001` para leer correctamente la salida de `netsh` con tildes, e interpretación de los códigos de salida de `robocopy`.
- **Soporte multiplataforma:** rutas de perfiles en Windows y Linux, incluidos los perfiles de navegadores instalados con snap repartidos en varias revisiones.
- **Autoinstalación y autoactualización:** el binario se copia a una ubicación fija, se registra en el inicio (registro `Run` + lanzador `.vbs` oculto en Windows, `systemd --user` en Linux) y consulta `version.json` cada 6 horas para reemplazarse. El script de publicación valida que la versión del paquete y la del binario coincidan.
- **Internacionalización y temas** con React Context, persistidos en `localStorage`.

## Integraciones

- **Sistema operativo:** Windows (PowerShell/CIM, registro, Spooler, netsh) y Linux (CUPS, NetworkManager, systemd).
- **Bases SQLite de los navegadores:** historial y cookies de Chromium, Firefox y GNOME Web.
- **GitHub Pages:** hospeda el panel, los instaladores, los binarios y el manifiesto de versión.
- **api.ipify.org:** consulta de la IP pública.
- **TeamViewer y AnyDesk:** apertura de las aplicaciones si están instaladas.

## Seguridad

- El agente solo escucha en `127.0.0.1`, así que no queda expuesto a la red.
- Las acciones destructivas piden confirmación en el panel, y el borrado se hace con el navegador cerrado para evitar corrupción parcial.
- Las acciones admitidas y las carpetas o aplicaciones abribles se limitan a listas cerradas, y los dominios se normalizan antes de usarse.
- Las contraseñas Wi-Fi se muestran ocultas por defecto.
- **Puntos a reforzar identificados:** la API local no tiene autenticación y acepta peticiones de cualquier origen (CORS abierto); algunos parámetros se interpolan en comandos de shell; y las actualizaciones no verifican firma ni hash.

## Retos técnicos

- Ejecutar acciones del sistema desde una aplicación web sin romper el modelo de seguridad del navegador.
- Borrar datos de navegadores con archivos bloqueados y procesos que se relanzan en segundo plano.
- Mapear la estructura interna de perfiles de 10 navegadores (Chromium, Gecko y WebKitGTK) en dos sistemas operativos.
- Imprimir en impresoras "a archivo" desde un servicio sin interfaz, evitando el diálogo "Guardar como".
- Reemplazar un ejecutable de Windows en uso durante la autoactualización.
- Distribuir un servicio Node.js a equipos sin Node.js instalado.

## Mejoras futuras recomendadas

- Restringir CORS al origen del panel y añadir un token compartido entre panel y agente.
- Validar con listas blancas todos los parámetros que llegan a comandos de shell.
- Firmar los binarios o verificar un hash en `version.json` antes de actualizar.
- Añadir pruebas automatizadas (unitarias para `commands.js`, de componentes para el panel) y CI.
- Hacer configurables las URLs del agente y limpiar los jobs terminados de la memoria.
- Generar los binarios de Windows y Linux en CI para evitar publicar uno solo.

## Estado del proyecto

- **Versiones:** panel `0.1.0` y agente `0.3.0`.
- **Despliegue:** configurado para GitHub Pages (`https://alvarosiles.github.io/browser-tools/`) mediante `npm run deploy`.
- **Madurez:** proyecto funcional en desarrollo activo (rama `dev`), sin pruebas automatizadas ni CI. No hay evidencia en el repositorio de uso en producción por parte de terceros.

## Capturas

Capturas de la aplicación real en ejecución (panel con Vite y agente local activo, tema oscuro, 1440 px de ancho a escala 1,5). El campo de dominio muestra `example.com` como dato de demostración.

| Archivo | Contenido |
|---|---|
| `presentacion/01-vista-principal.png` | Vista principal: cabecera con el estado del servicio ("Servicio activo"), personalización y la matriz de navegadores con los instalados detectados (punto verde). |
| `presentacion/02-funcionalidad-principal.png` | Módulo central: reset de perfil, borrado de datos y respaldo por navegador, y borrado de los datos de un dominio. |
| `presentacion/03-funcionalidad-complementaria.png` | Reparaciones automáticas: SFC, DISM, CHKDSK, Flush DNS, Reset Winsock y ejecución en un clic, con el aviso de que el agente no corre como administrador. |
