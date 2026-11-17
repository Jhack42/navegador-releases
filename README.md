# navegador

An Electron application with React and TypeScript

este proyecto funciona con pnpm

## Navegación web

La interfaz del navegador se encuentra en `src/renderer/src/components/browser`.
Las pestañas y controles React se comunican con el proceso principal mediante la
API tipada de `src/preload`. El proceso principal crea una `WebContentsView`
aislada por pestaña y actualiza sus límites según el área visible del renderer;
las páginas web no se cargan en un `iframe`.

La barra de direcciones acepta sitios web HTTP/HTTPS y las páginas internas
`navegador://history` y `navegador://home`, y envía el texto restante al motor
de búsqueda predeterminado. El historial y la página principal usan entradas
HTML y React independientes, servidas en una `WebContentsView` aislada. La
página principal separa componentes, hook, servicio y estilos bajo
`src/renderer/src/pages/home`; su preload interno expone solo navegación y
accesos directos tipados, sin APIs de Node.js. Los marcadores se guardan en el
almacenamiento local del perfil y las visitas del historial en una base SQLite
persistente. La página principal ofrece fondos animados, permite seleccionar
imágenes y videos locales y guarda los medios en IndexedDB del perfil (hasta
25 MB por imagen y 150 MB por video). Los videos se pausan al ocultar la página
y se respetan las preferencias de movimiento reducido. La configuración de
widgets tiene un registro extensible, todavía sin widgets incluidos.

## Actualizaciones automáticas

Las actualizaciones automáticas están habilitadas únicamente para la versión
instalada de Windows x64 creada con NSIS. Al iniciar, el proceso principal
espera tres segundos y comprueba GitHub Releases sin bloquear la ventana. Si
hay una versión nueva, la descarga ocurre en segundo plano; en **Menú → Acerca
del navegador** se muestran el estado, el progreso, la versión y las notas.
La instalación requiere que el usuario elija **Reiniciar e instalar**. No se
instala automáticamente al cerrar.

El proveedor está configurado en `electron-builder.yml` para
`Jhack42/navegador-releases`; el actualizador consulta ese repositorio público
sin token. El job de `.gitlab-ci.yml` solo acepta etiquetas y valida que
`package.json` coincida exactamente con `v<versión>`. La compilación de
electron-builder siempre usa `--publish never`; después, un único publicador
idempotente de la API de GitHub procesa el instalador NSIS x64 y sus metadatos.
El job se serializa con `resource_group`, crea o reutiliza una sola Release,
comprueba versión, nombres, SHA-512 de `latest.yml`, tamaños y SHA-256 de los
tres assets (`Navegador-Setup-<versión>.exe`, su `.blockmap` y `latest.yml`) y
solo entonces publica la Release. Si encuentra varios objetos Release para la
misma etiqueta, falla sin borrarlos ni alterarlos para evitar una decisión
destructiva automática. Los comandos locales de build también desactivan la
publicación.

`GH_TOKEN` solo se usa en el job protegido de GitLab para publicar. No se
empaqueta ni se expone a renderer o preload; la aplicación instalada no lo
necesita para consultar Releases públicas. No se publica el repositorio fuente.
GitHub no incluye las Releases Draft en el feed público que lee
`electron-updater`; una versión solo estará disponible después de que los tres
assets hayan sido verificados y la Release se publique.

### Requisitos de GitLab CI/CD

El repositorio contiene el pipeline, pero no su configuración de runner; por
ello no se puede confirmar qué runners están registrados en GitLab. Antes de
publicar, registra o asigna un runner **Windows** con el tag `windows`, shell
PowerShell, Node.js 22.12 o posterior, Git, Corepack/pnpm 10.18.1 y acceso de
red a GitLab y GitHub. Este job no está preparado para un runner Linux/macOS ni
para compilación cruzada de NSIS.

En **Settings → CI/CD → Variables**, configura `GH_TOKEN` como variable
enmascarada, oculta y protegida, sin imprimirla en scripts. Protege la rama de
publicación (por ejemplo `main`) y el patrón de etiquetas `v*`, limitando quién
puede crear etiquetas protegidas. La variable protegida solo se entrega a esos
pipelines. El token debe conservar acceso de escritura a Contents únicamente
para `Jhack42/navegador-releases`.

### Publicar y verificar

1. Actualiza `version` en `package.json` y confirma el cambio en GitLab.
2. Después de actualizar la versión, crea y envía la etiqueta que coincida
   exactamente con ella. Por ejemplo, para `1.0.4`:

   ```powershell
   git tag v1.0.4
   git push origin v1.0.4
   ```

3. En GitLab, comprueba el job `publish-windows-nsis`. Una discrepancia entre
   la etiqueta y `package.json`, la falta de `GH_TOKEN` o un fallo de
   compilación detiene el job antes de publicar.
4. Comprueba que exista exactamente una Release **publicada**, no Draft, con
   `Navegador-Setup-<versión>.exe`, `Navegador-Setup-<versión>.exe.blockmap` y
   `latest.yml`. El job no termina correctamente si falta un asset o su tamaño /
   SHA-256 no coincide. Si avisa que hay Releases duplicadas para la etiqueta,
   revisa y resuelve esas Releases manualmente antes de volver a ejecutar el job.
5. Para una prueba de actualización, instala primero una versión NSIS anterior;
   publica una versión superior y abre **Menú → Acerca del navegador → Buscar
   actualizaciones**. Verifica la descarga y usa **Reiniciar e instalar** para
   completar el ciclo. No sirve una ejecución de desarrollo ni la versión
   portable para esta prueba.

La versión portable no usa `electron-updater`: sigue conservando
`Navegador-data` junto al ejecutable y debe actualizarse manualmente. No se
reemplaza el perfil durante la actualización NSIS; el desinstalador existente
también conserva los datos de usuario.

Un futuro actualizador portable debe ejecutarse fuera del proceso del navegador:
descargar en una carpeta temporal del mismo volumen, verificar la firma del
ejecutable contra una identidad de publicador fijada en la aplicación, esperar
al cierre, conservar una copia de respaldo y sustituir el ejecutable con
recuperación si el reemplazo falla. Después debe relanzar el navegador sin
modificar `Navegador-data`. El `SHA-512` publicado junto al archivo no basta
para autenticar el publicador; esa función requiere firma digital verificable.
Este mecanismo portable no está implementado.

Los hashes SHA-512 de `latest.yml` detectan daños o inconsistencias en los
archivos, pero no autentican por sí solos al publicador. Para una distribución
pública segura, firma el instalador con un certificado de firma de código y
configura `CSC_LINK` y `CSC_KEY_PASSWORD` como variables protegidas de GitLab;
además, configura en electron-builder el nombre exacto del publicador del
certificado para que el actualizador pueda verificarlo. No se ha configurado
ningún certificado en este repositorio, así que las compilaciones actuales no
deben considerarse autenticadas por firma digital.

## Grupos de pestañas

Haz clic derecho en una pestaña para crear un grupo o moverla a uno existente.
También puedes arrastrar pestañas entre sí o sobre una etiqueta de grupo. El
menú contextual de cada grupo permite cambiar su nombre y color, contraerlo,
desagruparlo o cerrar todas sus pestañas. El menú del botón «Nueva pestaña»
permite crear un grupo con una pestaña nueva.

La organización, el orden, el estado contraído y las URL se guardan desde el
proceso principal en `browser-state.json`, dentro de `app.getPath('userData')`.
Al reiniciar, las pestañas se restauran bajo demanda; contraer grupos o mover
pestañas no destruye ni reemplaza sus `WebContentsView`.

## Reproducción automática

En **Configuración → Privacidad y contenido → Reproducción automática** puedes
permitir todo autoplay, permitir solo contenido silenciado o exigir interacción.
La política se configura con `webPreferences.autoplayPolicy` de Chromium y la
preferencia se guarda en el mismo `browser-state.json`.

Al cambiarla, las vistas abiertas se recrean tras confirmarlo. Electron restaura
las entradas de navegación y el estado de página que Chromium tenga disponible;
las vistas nuevas siguen usando la misma sesión del perfil. El estado temporal
no guardado de un sitio podría perderse. La política no anula restricciones del
sitio, permisos de iframe, códecs, red ni controles del usuario.

## Importar datos

En **Configuración → Importar datos del navegador** se detectan Google Chrome
Stable, Beta, Dev y Chromium, incluidos sus perfiles. La vista previa muestra
las categorías disponibles antes de confirmar. Los marcadores, el historial,
los motores de búsqueda compatibles y la página de inicio se integran con el
navegador; los datos de autocompletado se guardan cifrados mediante `safeStorage`
del sistema operativo (el navegador todavía no los aplica automáticamente a los
formularios). Las bases de datos del perfil de origen se abren en modo solo
lectura y los datos permanecen en el equipo.

Las contraseñas no se importan porque el navegador aún no tiene una bóveda
compatible con el descifrado autorizado del sistema operativo. Tampoco se copian
cookies ni pestañas abiertas; estas últimas usan un formato de sesión interno de
Chrome. Los errores parciales y las categorías no disponibles se muestran en la
interfaz.

## Extensiones

En una página de detalles compatible de Chrome Web Store aparece un botón del
navegador para preparar la instalación. También puedes abrir **Extensiones**
para elegir una carpeta desempaquetada o un paquete CRX3 local. Cada instalación
requiere confirmación; antes se muestran los permisos y las advertencias del
manifiesto. Las carpetas locales se marcan como modo desarrollador porque no
tienen una firma CRX verificable.

El paquete CRX3 se verifica y extrae en el proceso principal. Las extensiones
administradas se guardan bajo `app.getPath('userData')/extensions` y se
restauran en la sesión persistente al iniciar. La descarga de Web Store usa el
servicio de actualización de Chromium, que no es una API pública estable. No se
intercepta ni se ejecuta el botón del sitio web. La barra propia muestra los
iconos de `manifest.json`, guarda qué extensiones se fijaron y permite ordenar
los iconos. Abre `action.default_popup` o `browser_action.default_popup` como una
página `chrome-extension://` en una ventana aislada que comparte la sesión de
Electron. Las opciones del manifiesto se abren en una ventana separada.

Electron 44 solo documenta un subconjunto de las APIs de Chrome: `runtime`,
`storage.local`, `scripting` y parte de `tabs` están disponibles; `storage.sync`
y `storage.managed` son parciales. Electron no documenta `chrome.action`,
`browserAction`, `permissions`, `contextMenus`, `notifications` ni `commands`.
La barra del navegador implementa el icono, el popup estático, las opciones y
las preferencias de fijado, pero no emula esas APIs de ejecución. El menú solo
muestra permisos declarados; no puede concederlos por sitio, y la coincidencia
de host indica una declaración del manifiesto, no un permiso efectivo. Por ello
no todas las extensiones de Chrome Web Store son compatibles.

Consulta la [lista oficial de APIs de extensiones de Electron](https://www.electronjs.org/docs/latest/api/extensions)
antes de instalar una extensión que dependa de APIs no listadas. Las extensiones
Manifest V3 también pueden fallar si sus service workers usan APIs incompatibles.

## Recommended IDE Setup

- [VSCode](https://code.visualstudio.com/) + [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint) + [Prettier](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode)

## Project Setup

### Install

```bash
$ pnpm install
```

### Development

```bash
$ pnpm dev
```

### Build

```bash
# Windows NSIS installer
$ pnpm run build:win

# Portable Windows x64
$ pnpm run build:portable

# Both Windows distributions (compiles electron-vite once)
$ pnpm run build:win:all

# For macOS
$ pnpm run build:mac

# For Linux
$ pnpm run build:linux
```

The NSIS installer is written to `release/Navegador-Setup-<version>.exe` and
supports per-user installation, choosing the install directory, and desktop and
Start menu shortcuts. Uninstalling asks whether to delete the profile; the
default is to keep it. `build:win:all` produces both the installer and portable
executable after one application build.

### Windows portable profile

`pnpm run build:portable` creates a portable x64 executable under `release/`.
On its first launch, it creates `Navegador-data/` next to the executable and
stores the browser profile there, including history, settings, extensions and
Chromium session data/cache. Keep that folder beside the executable and make
sure the drive is writable. Installed builds continue to use their normal
Windows profile directory.

Some data encrypted by Windows or Chromium may be bound to the Windows account
or computer and may not be usable after moving the portable profile to another
PC or account.
