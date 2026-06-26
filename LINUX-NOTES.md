# LINUX-NOTES.md — Build & runtime notes (this fork, on Linux)

> Materia prima de los Sprints 2 (captura de audio) y 4 (modelo por defecto).
> Máquina de referencia del Sprint 0/1. Los nombres de dispositivos son **específicos de esta máquina y del routing vigente** — descubrir en runtime, no hardcodear.

## Sistema
| | |
|---|---|
| Distro | **openSUSE Tumbleweed** (rolling) |
| Package manager | **zypper** |
| Sesión gráfica | **X11** (`XDG_SESSION_TYPE=x11`, `DISPLAY=:0`) — el atajo global del Sprint 2 es viable; en Wayland sería limitado |
| Servidor de audio | **PipeWire 1.6.4** con capa de compat **PulseAudio** (`Server Name: PulseAudio (on PipeWire 1.6.4)`) |

## Hardware detectado (lo detecta el producto, no lo pregunta — Sprint 4)
| | |
|---|---|
| CPU | **AMD Ryzen 5 PRO 4650U** (6 núcleos / 12 hilos) |
| RAM | 30 GiB |
| GPU | AMD Radeon Vega (Renoir, integrada) — **sin NVIDIA/CUDA, sin ROCm, sin Vulkan SDK** |
| Aceleración whisper | **CPU-only** (build sin features). La iGPU AMD no acelera whisper estándar |

**Implicancia:** el build cae en `platform-default` → CPU puro (whisper-rs sin BLAS). Para vivo conviene `base`/`small` int8. Evaluar whisper.cpp/Parakeet con más núcleos en Sprint 4.

## Dispositivos de audio (Sprint 0 — `pactl`)
- **Default Sink:** `bluez_output.DC_C4_9C_40_85_FB.1` (auriculares Bluetooth).
- **Monitor del sistema (lo que se escucha / "los demás"):** `bluez_output.DC_C4_9C_40_85_FB.1.monitor` — termina en `.monitor`, capturable con `ffmpeg -f pulse -i <monitor>` o cpal.
- **Mic (vos):** `alsa_input.pci-0000_06_00.6.analog-stereo`.
- **Importante:** el monitor correcto = `<Default Sink>.monitor`, y el Default Sink **cambia** si se desconectan los BT (pasaría a `alsa_output.pci-0000_06_00.6.analog-stereo.monitor`). Resolver con `pactl get-default-sink` + escuchar `pactl subscribe` para re-rutear en caliente.
- El código del repo ya trae `audio/devices/platform/linux.rs` → `configure_linux_audio()` que enumera devices y marca las fuentes con `"monitor"` en el nombre como *System Audio*. Base lista para Sprint 2.

## Arquitectura real del repo (cambió respecto al brief)
- App **Tauri 2.x** (Rust + Next.js 14 + React 18) en `frontend/`. Todo (captura, transcripción, persistencia, summaries) vive en el core Rust/Tauri.
- El **backend Python/FastAPI en `backend/` está ARCHIVADO y sin soporte** (ver CLAUDE.md). **No hay backend separado que levantar** — un proceso menos que el brief asumía.
- Sidecars: **`llama-helper`** (crate en la raíz, usa `llama-cpp-2`) para LLM local, y **ffmpeg** (se descarga en build). Ambos van como `externalBin` de Tauri.

## Versiones
- Tauri **2.6.2** (`@tauri-apps/cli ^2.1.0`, `@tauri-apps/api ^2.6.0`).
- Rust **1.96.0** (rustup, toolchain stable).
- Node **v24.14.1**, pnpm **11.1.2**, Python **3.13**.

## Dependencias de sistema instaladas (zypper)
```bash
sudo zypper install -y \
  'pkgconfig(webkit2gtk-4.1)' \   # motor web de la ventana Tauri 2  (instaló 2.52.4)
  'pkgconfig(gtk+-3.0)' \         # GTK 3 (3.24.52)
  'pkgconfig(librsvg-2.0)' \      # SVG (2.62.3)
  'pkgconfig(ayatana-appindicator3-0.1)' \  # tray icon, Sprint 2 (0.5.93)
  'pkgconfig(alsa)' \            # ALSA dev para cpal/captura (1.2.16)
  clang clang-devel llvm-devel \  # libclang para bindgen de whisper-rs (clang 22)
  cmake gcc-c++ git
```
> openSUSE provee webkit2gtk-4.1 vía `pkgconfig(webkit2gtk-4.1)` (Tauri v2). En Tauri v1 sería 4.0.

## Comandos de build que funcionaron
```bash
# 1) Toolchain Node
cd frontend && pnpm install

# 2) Build + run (dev, con hot reload). Detecta GPU (acá → CPU), compila llama-helper,
#    lo copia a src-tauri/binaries/llama-helper-<triple>, y lanza tauri dev:
./dev-gpu.sh
# (equivalente CPU directo, sin sidecar GPU: pnpm run tauri:dev:cpu — pero igual requiere el sidecar)
```

## Fixes de Linux que hicieron falta
1. **`dev-gpu.sh` / `build-gpu.sh` — detección de OS rota en openSUSE.**
   Chequeaban `[[ "$OSTYPE" == "linux-gnu"* ]]`, pero en este bash `$OSTYPE == "linux"` (sin `-gnu`) → abortaba con *"Unsupported OS: linux"*.
   **Fix:** cambiar el patrón a `[[ "$OSTYPE" == linux* ]]` (2 ocurrencias por script). Aplicado en ambos.
2. **Sidecar `llama-helper` ausente.** `cargo build` directo de `src-tauri` falla con
   `resource path 'binaries/llama-helper-x86_64-unknown-linux-gnu' doesn't exist`.
   No es un fix de código: hay que **construir el sidecar primero** (lo hace `dev-gpu.sh`). Una vez copiado a `src-tauri/binaries/`, `cargo build` suelto ya funciona.
3. **`whisper-rs` no compila — bindgen + libclang 22 generan un struct opaco.**
   Con clang 22 (Tumbleweed, bleeding edge), `whisper-rs-sys 0.11.1` corre bindgen y genera
   `whisper_full_params` como tipo **incompleto/opaco** (`{ _address: u8 }`), lo que rompe la
   compilación del propio `whisper-rs` con **71× `error[E0609]: no field ... on type whisper_full_params`**
   (`greedy`, `beam_search`, `n_threads`, `translate`, …).
   Diagnóstico: `clang -fsyntax-only wrapper.h` parsea **sin error** → el fallo es de bindgen+libclang22, no del header.
   **Fix:** el crate trae un `src/bindings.rs` pre-generado correcto; se usa con el env var
   `WHISPER_DONT_GENERATE_BINDINGS=1`. Persistido en **`.cargo/config.toml`** (`[env]`) en la raíz del repo,
   así toda invocación de cargo (tauri dev, build, sidecar) lo toma. Forzar regen una vez con
   `cargo clean -p whisper-rs-sys -p whisper-rs`.
   *Alternativa futura:* bumpear `whisper-rs` a una versión con bindgen compatible con clang ≥20.

## Resultado del Sprint 1
- ✅ `cargo build` (CPU) compila limpio (solo warnings de dead-code) tras los 3 fixes.
- ✅ La ventana **meetily** abre en X11 y la UI (Next.js/React) renderiza: barra de navegación lateral con Home, **Recording (mic)**, **Import (upload)**, Summary/Notes, Settings, About.
- ✅ Sin errores de runtime en el arranque. `next dev` en `localhost:3118`, binario Tauri levantado.
- Verificación visual por screenshot de la ventana (no se automatizó el click por pestañas para no interrumpir la sesión activa del usuario; `xdotool` no está instalado).

## Cómo correr: estable (estático) vs desarrollo (hot reload)
- **Estable / demo (recomendado para revisar):** build estático embebido, sin dev-server.
  ```bash
  cd frontend && pnpm tauri build --debug --no-bundle   # reusa binario debug, ~1-2 min
  ./../target/debug/meetily                              # ejecutable (ventana "Soferly")
  ```
  El frontend queda pre-compilado en `frontend/out/` y embebido → no hay `ChunkLoadError`.
- **Desarrollo (hot reload):** `cd frontend && ./dev-gpu.sh`. Útil para iterar, **pero** ver el gotcha de abajo.

## Gotcha: `ChunkLoadError` en modo dev dentro del webview
En `tauri dev`, Next compila las rutas **bajo demanda**; la primera carga tarda (~10s) y el webview de Tauri (webkit2gtk) corta la carga del chunk antes de tiempo → `ChunkLoadError: Loading chunk app/layout failed (timeout ...:3118/...)`. Síntoma: la UI aparece pero **los clicks no hacen nada** (el JS no hidrató: cáscara estática muerta), y a veces aparece el overlay rojo de error.
- **No es un bug del código.** Es la fragilidad del dev-server dentro del webview en Linux.
- **Workaround inmediato:** usar el build estático (arriba). Para Sprint 2 (que requiere iterar), evaluar: recargar el webview tras el primer compile, subir el timeout de carga de chunks, o pre-compilar rutas. Anotar como deuda de DX.
- **OJO:** no matar el `next dev` (puerto 3118) con una ventana abierta apuntándole → provoca el mismo `ChunkLoadError`.

## Rebrand a Soferly (hecho)
- `productName` y título de ventana → **Soferly** (en `tauri.conf.json`). `identifier` se dejó `com.meetily.ai` para no cambiar rutas de datos (cambiarlo a `com.soferly.app` implica empezar con datos limpios).
- Todos los textos visibles `Meetily`→`Soferly` (sidebar, About, Info, onboarding, permisos, welcome). Quedaron sin tocar IDs/URLs internos (`meetily_user_id`, `MeetilyRecoveryDB`, links a zackriya/github, rutas Homebrew de macOS).
- **Logo nuevo:** `frontend/public/soferly-logo.svg` (onda de sonido sobre cuadrado redondeado, gradiente índigo→violeta). PNGs de la app (`logo.png`, `logo-collapsed.png`) y todo el set de íconos del sistema regenerados con `pnpm tauri icon`. SVG→PNG con `cairosvg` (ImageMagick tiene el coder SVG bloqueado por policy en esta máquina).

## Gotchas para Sprint 2
- **Wayland vs X11:** esta sesión es X11 → atajo global OK. Si se cambia a Wayland, el global shortcut puede no funcionar; anotarlo.
- **Tray** depende de `ayatana-appindicator3` (ya instalado).
- **Default sink cambiante** (BT ↔ parlantes): seguir `get-default-sink` + `pactl subscribe`.
- El monitor mezcla TODO lo que va al sink (música incluida): para reuniones, rutear solo la app de la llamada o documentar el comportamiento.
