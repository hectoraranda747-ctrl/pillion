# Compilar Pillion para iPhone desde Windows

GitHub Actions compila en un Mac remoto. No necesitas un Mac local ni credenciales
de Apple en GitHub. La instalación y la firma de Apple se realizan en Windows.

## 1. Crear y preparar tu fork

1. Inicia sesión en GitHub y abre [alexandrevega/pillion](https://github.com/alexandrevega/pillion).
2. Pulsa **Fork**, selecciona tu cuenta y pulsa **Create fork**.
3. Añade a tu fork estos dos archivos, conservando las rutas desde la raíz:
   `.github/workflows/build-ios.yml` y `docs/WINDOWS-IOS-BUILD.md`.
   Puedes usar **Add file → Create new file** y copiar su contenido, o Git desde Windows.
4. Confirma los cambios en la rama predeterminada (`main` en el original analizado).
   Si trabajas en otra rama, abre un pull request y fusiónalo primero: GitHub necesita
   el workflow en la rama predeterminada para mostrar el disparador manual.

No subas `Signing.xcconfig`, certificados, perfiles, Apple ID ni contraseñas.
El workflow crea un `Signing.xcconfig` vacío únicamente en el runner.

## 2. Activar Actions y generar la IPA

1. En **tu fork**, abre **Actions**.
2. Si aparece, pulsa **I understand my workflows, go ahead and enable them**.
   Si Actions está deshabilitado, revisa **Settings → Actions → General** y permite
   GitHub Actions y las acciones oficiales `actions/checkout`, `actions/setup-java`
   y `actions/upload-artifact`. Una política de organización puede requerir al administrador.
3. En la columna izquierda selecciona **Build iOS unsigned IPA**.
4. Pulsa **Run workflow**, selecciona `main` (o la rama que quieres compilar) y pulsa
   el botón verde **Run workflow** del desplegable.
5. Abre la ejecución y espera a que el trabajo `build-ios` termine en verde.
   La primera compilación descarga Kotlin/Native y dependencias y puede tardar bastante.

El workflow solo se inicia manualmente. En repositorios privados comprueba la cuota
y las condiciones de uso de runners macOS de tu cuenta.

## 3. Descargar Pillion.ipa

En el resumen de la ejecución, baja hasta **Artifacts** y pulsa
**Pillion-iOS-unsigned**. Debes estar conectado a GitHub.
Extrae el ZIP descargado: dentro encontrarás **Pillion.ipa**.
No entregues el ZIP del artifact a Sideloadly ni descomprimas la propia IPA para instalarla.
Los artifacts se conservan durante 14 días; después puedes ejecutar otra build.

**Pillion-iOS-build-logs** contiene el registro, versiones de herramientas, el SHA-256
de la IPA cuando se ha verificado y el `Package.resolved` generado si está disponible.
Si falla la build, abre el primer paso rojo y consulta ese artifact; la IPA solo se
publica si su compilación y todas las comprobaciones terminan correctamente.

## 4. Firmar e instalar con Sideloadly en Windows

1. Descarga [Sideloadly](https://sideloadly.io/) desde su sitio oficial. Sigue sus
   instrucciones para instalar las versiones web de iTunes e iCloud que requiere en Windows.
2. Conecta el iPhone por USB, desbloquéalo y acepta **Confiar en este ordenador**.
3. Abre Sideloadly, selecciona el iPhone y arrastra **Pillion.ipa** a su ventana.
4. Usa el modo de instalación con Apple ID e introduce tu cuenta **solo en Sideloadly**.
   Completa la autenticación que solicite. No se configura ningún secreto en Actions.
5. Conserva las extensiones: no actives **Remove Extensions** ni elimines
   `PillionBroadcast.appex`. Evita cambiar manualmente los identificadores; el selector
   de Pillion busca el identificador de la app terminado en `.broadcast`.
6. Pulsa **Start**. Sideloadly debe volver a firmar tanto la app como la extensión.
7. Si iOS lo solicita, confía en el perfil en **Ajustes → General → VPN y gestión
   de dispositivos** y activa **Modo de desarrollador** en **Privacidad y seguridad**
   cuando sea necesario, siguiendo el reinicio y confirmación del iPhone.
8. Abre Pillion y pulsa **Start mirroring → Pillion Mirror → Start Broadcast**.
   Comprueba que aparece la extensión y que la emisión se mantiene al cambiar de app.

Con una cuenta gratuita, las apps caducan a los **7 días**: repite la firma antes de
caducar o configura la renovación automática de Sideloadly. Habitualmente hay un
límite de **3 apps instaladas** y **10 App IDs por 7 días**; la gestión de IDs de
extensiones depende del firmador. Consulta la [FAQ oficial](https://sideloadly.io/faq).

## Qué compila y qué significa «unsigned»

La base revisada es el commit `6647af22f035ad74dc98f0e2a29e0c8a769a1caa` del original.
No tenía workflows: `.github/` solo contenía `FUNDING.yml`. Las fuentes actuales
incluyen iOS aunque algunas partes del README todavía lo describen como futuro.
Se revisaron [IOS.md](IOS.md), [IOS-SIDELOAD.md](IOS-SIDELOAD.md), el changelog y la
documentación de arquitectura e integración SDL.

El workflow reutiliza **`iosApp/build-ipa.sh Release`** sin modificarlo. Este genera
`iosApp/iosApp.xcodeproj` con XcodeGen a partir de `iosApp/project.yml`, utiliza el
scheme `iosApp` y el destino **`generic/platform=iOS`**, y compila el framework
estático `ComposeApp` mediante las fases Gradle existentes. Xcode resuelve
SmartDeviceLink mediante Swift Package Manager; no se necesita CocoaPods.
Java 17 y el Android SDK del runner permiten configurar el módulo multiplataforma.

La build desactiva la firma de Apple con `CODE_SIGNING_ALLOWED=NO`,
`CODE_SIGNING_REQUIRED=NO` y `CODE_SIGN_IDENTITY=""`. El script después aplica
**sellos locales ad hoc** (`codesign --sign -`), primero a la extensión y luego a
la app. No usan certificados, perfiles ni claves privadas de Apple. Se conservan
porque el script los incorpora para evitar problemas al volver a firmar la extensión.
La IPA no está completamente libre de firmas binarias y **no es instalable directamente**
en un iPhone normal: Sideloadly debe darle una firma válida para ese dispositivo.

La verificación del workflow abre la IPA y exige:

- `Payload/iosApp.app` y `PlugIns/PillionBroadcast.appex`, con ejecutables ARM64
  de iPhoneOS, versiones coincidentes e identificadores relacionados.
- Registro ReplayKit `com.apple.broadcast-services-upload`, clase
  `PillionBroadcast.SampleHandler` y modo `RPBroadcastProcessModeSampleBuffer`.
- Recursos de Compose, sellos ad hoc válidos y ausencia de provisioning profiles.

Estas comprobaciones prueban el empaquetado; no sustituyen una prueba de ReplayKit
en un iPhone después de que Sideloadly haya vuelto a firmar. No se modifica Android,
NaviLite, Bluetooth ni la implementación de ReplayKit.

## Limitaciones y reproducibilidad

Se fija **macos-15**, **Xcode 16.0** (compatible con el Kotlin 2.1.0 del proyecto) y
**XcodeGen 2.42.0**, cuya descarga se comprueba con SHA-256. Si GitHub retira ese
Xcode, el workflow fallará explícitamente: habrá que revisar juntos Kotlin y Xcode.
Consulta la [compatibilidad de Kotlin](https://kotlinlang.org/docs/multiplatform/multiplatform-compatibility-guide.html)
y el [inventario del runner](https://github.com/actions/runner-images/blob/main/images/macos/macos-15-Readme.md).

Es un proceso repetible, pero no una garantía de binarios idénticos: la imagen de
GitHub y Java 17 reciben actualizaciones y el proyecto original permite versiones
de SmartDeviceLink desde 7.6.1, con dependencias transitivas. Se conserva esa
configuración y se guarda la resolución SPM para diagnosticar cambios. El Gradle
8.11.1 y AGP 8.7.3 originales también están por encima del rango de compatibilidad
plena publicado para Kotlin 2.1.0; no se cambia el sistema de build compartido de
Android sin un fallo concreto que lo justifique.

**App Groups:** ambos targets declaran `group.app.pillion`. Aunque `IOS.md` dice
que no hace falta, el código actual de `Shared/Transport.swift` lo consulta para
ajustes de calidad y FPS. La build sin credenciales puede compilarlo; la firma
ad hoc existente no conserva esos entitlements. La capacidad disponible al instalar
depende del perfil y del firmador, y no se debe dar por disponible con una cuenta
gratuita. Sin acceso al grupo, el código ya usa 15 FPS y calidad JPEG 0.4 como
valores predeterminados. No se elimina ni reescribe esa lógica.

No se ha ejecutado Xcode desde Windows ni se ha probado la IPA en un iPhone en esta
preparación. La primera ejecución en tu fork confirmará la compilación real; una
build correcta tampoco acredita la conexión física a una moto.

## 5. Actualizar desde el repositorio original

1. En la página **Code** de tu fork pulsa **Sync fork → Update branch**.
2. Conserva los dos archivos de esta guía. Si hay conflictos, usa la opción de
   abrir un pull request, revísalos y resuélvelos antes de fusionar; no descartes tus cambios.
3. Revisa si el original cambió `iosApp/build-ipa.sh`, `iosApp/project.yml`, los
   nombres de los targets, Kotlin o sus instrucciones iOS. Si el original añade
   un workflow con el mismo nombre, combina los cambios deliberadamente.
4. Ejecuta de nuevo **Actions → Build iOS unsigned IPA → Run workflow**.
5. Descarga la nueva IPA y vuelve a instalarla con la misma cuenta y configuración
   de identificadores en Sideloadly para actualizar la app existente.

GitHub explica [cómo ejecutar workflows manuales](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow)
y [cómo sincronizar un fork](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/syncing-a-fork).
