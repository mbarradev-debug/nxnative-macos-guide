# nxnative en Mac: de un juego de Switch a una app

*English version: [README.md](README.md)*

Guía para convertir cualquier juego de Switch o Switch 2 en una app de Mac con [nxnative](https://github.com/seedds/nxnative). Solo Mac, sin iOS.

Instalación de ejemplo usada en esta guía (ajusta las rutas a las tuyas):

| Qué | Dónde |
|---|---|
| Repositorio | `~/nxnative` |
| Python con las dependencias (venv) | `/Volumes/BACKUP/venv` |
| Juegos (`.nup` / `.nsp`) | `/Volumes/BACKUP/Games/<juego>/` |
| Llaves (solo para juegos de Switch 1) | `~/keys/prod.keys` |
| Copias de las partidas | `~/Documents/nxnative-saves` |

---

## 0. Requisitos (una sola vez)

1. **Mac con Apple silicon** y macOS 26.6 o posterior.
2. **Xcode 27.2** en `/Applications/Xcode.app`, seleccionado como herramienta activa:
   ```
   sudo xcode-select -s /Applications/Xcode.app
   xcodebuild -downloadComponent MetalToolchain
   ```
   Si no quieres usar `sudo`, antepone `DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer` a cada comando de build.
3. **Herramientas de Homebrew:**
   ```
   brew install xcodegen zstd glslang
   ```
4. **Python con las dependencias**, en un venv en el disco externo. No uses `pip install` directo, porque Homebrew lo bloquea.
   ```
   python3 -m venv /Volumes/BACKUP/venv
   /Volumes/BACKUP/venv/bin/pip install -r ~/nxnative/.github/requirements/tools.txt
   ```
   Si pip muestra avisos `Ignoring invalid distribution -xxx`, son inofensivos: vienen de los archivos `._*` que macOS crea en discos exFAT. Para limpiarlos: `dot_clean -m /Volumes/BACKUP/venv/lib`.

---

## 1. Antes de cada juego

- **Actualiza el repositorio:** `cd ~/nxnative && git pull`
- **Comprueba que el juego esté soportado:** busca su title ID (aparece en el nombre del archivo, por ejemplo `[0400486024DEC000]`) en `~/nxnative/dm/games/README.md`. Si no aparece, el script intenta un build "provisional" que puede no arrancar.
- **Lee la nota del juego,** `~/nxnative/dm/games/<titleID>/README.md`. Dice si funciona en Mac y qué falta.
- **Deja el juego en el disco externo,** nunca en tu carpeta de usuario.
- **Revisa el espacio libre:** `df -h ~`. Un juego grande necesita unos **70 GB libres en la Mac**:

  | Qué | Tamaño aproximado |
  |---|---|
  | Archivos extraídos del juego | ≈ tamaño del juego |
  | Shaders compilados | 10–15 GB |
  | La app final | ≈ tamaño del juego + 10 GB |

  Si te quedas sin espacio, el build falla. Peor aún, el disco del sistema se llena.

---

## 2. Construir la app de Mac

```
cd ~/nxnative
/Volumes/BACKUP/venv/bin/python3 scripts/build_ipa_from_nsp.py \
  "/Volumes/BACKUP/Games/<juego>/<archivo>.nup" \
  --platform mac-app \
  --work-dir ~/Games/<titleID>_build \
  -o ~/Games/"<Nombre del juego>.app"
```

- `--platform mac-app` genera una app de Mac en vez de un IPA de iOS. **No lo olvides.**
- `--work-dir` es la carpeta de trabajo: archivos extraídos y shaders compilados. Conviene que esté en el disco interno, que es más rápido. Si la conservas, un rebuild del mismo juego tarda minutos en vez de horas.
- `-o` es dónde queda la app. Ponla en el disco interno para que el juego cargue rápido.
- **Juegos de Switch 1** (`.nsp`, o una carpeta con NSPs): agrega `-k ~/keys/prod.keys`. Los `.nup` de Switch 2 no necesitan llaves.
- Siempre usa el Python del venv (`/Volumes/BACKUP/venv/bin/python3`), no `python3` a secas.

**Opciones útiles:**

| Opción | Para qué |
|---|---|
| `--icon imagen.png` | Ícono de la app, para cuando el juego no trae uno |
| `--seen-shaders` | Compila solo los shaders ya probados: minutos en vez de horas. Algunas zonas pueden verse incompletas |
| `-l es` | Idioma del juego, si lo tiene |
| `--help` | Todas las opciones |

**Cuánto tarda** (referencia: Ocarina of Time, 24 GB, en un M4):

| Paso | Primera vez | Rebuild con la carpeta de trabajo |
|---|---|---|
| Desempaquetar | ≈ 2 min | se reutiliza |
| Shaders de Mac | ≈ 45 min | se reutiliza |
| Pipeline archive (gráficos para tu GPU) | ≈ 27 min | se reutiliza |
| Empaquetar y firmar la app | ≈ 5 min | ≈ 5 min |

Para dejarlo corriendo y ver el avance desde otra ventana:
```
... comando de arriba ... > ~/Games/build.log 2>&1 &
tail -f ~/Games/build.log
```
Las líneas que empiezan con `[RED]` son errores. El build termina con una línea `done: <ruta de la app>`.

---

## 3. Jugar

- Abre la `.app` desde Finder. La primera vez tarda más en arrancar.
- En la pantalla de ajustes eliges frame rate, resolución, modo y otras opciones, y le das a **Play**. Mantén pulsado Option al abrir la app para volver a ver esa pantalla.
- Funcionan los mandos DualSense, Xbox y Pro Controller, y por Bluetooth también los Joy-Con 2 y el Pro Controller 2.

**Dónde queda cada cosa** (`<titleID>` es el del juego):

| Qué | Dónde |
|---|---|
| Partida y ajustes | `~/Library/Application Support/nxnative/<titleID>/` |
| Log, para revisar errores | `~/Library/Logs/nxnative/<titleID>.log` |
| Variables de configuración | `~/Library/Application Support/nxnative/<titleID>/env.txt` |

---

## 4. Copia de las partidas en Documentos

La app trae una función de sincronizar partidas que acepta cualquier carpeta. Para que deje una copia en `~/Documents/nxnative-saves`:

```
mkdir -p ~/Documents/nxnative-saves
echo "DM_SAVE_SYNC=$HOME/Documents/nxnative-saves" >> ~/Library/Application\ Support/nxnative/<titleID>/env.txt
```

- Usa la ruta absoluta: `~` no funciona dentro de `env.txt` (el `$HOME` entre comillas dobles lo expande la shell antes de escribir).
- La carpeta `~/Library/Application Support/nxnative/<titleID>/` existe recién después de abrir el juego una vez. Si aún no existe, créala con `mkdir -p`.
- Empieza a funcionar la próxima vez que abras el juego.
- Cada juego deja un solo archivo, `~/Documents/nxnative-saves/<titleID>/save.dmsync`, con la partida completa. Se actualiza unos segundos después de cada guardado y al cerrar el juego.
- Si macOS pregunta si la app puede acceder a Documentos, responde que sí.
- Antes de reemplazar una partida, la app guarda la anterior en `save-backups` y conserva las 10 más recientes.

---

## 5. Limpiar después

- **Cuando la app funcione, borra la carpeta de trabajo** (`~/Games/<titleID>_build`) para liberar entre 25 y 40 GB. La app no depende de ella. La desventaja es que un rebuild vuelve a tardar horas.
- Conserva el `.nup` o `.nsp` original en el disco externo, por si tienes que volver a construir la app.
- La app (`.app`) es lo único necesario para jugar.

---

## Problemas encontrados

| Error | Solución |
|---|---|
| `xcodebuild requires Xcode, but active developer directory ... CommandLineTools` | `sudo xcode-select -s /Applications/Xcode.app`, o usa `DEVELOPER_DIR=...` (ver paso 0) |
| `error: externally-managed-environment` (pip) | Instala en el venv, no en el Python del sistema |
| `[RED] the current Python ... is missing modules` | Corriste el script con `python3`; usa `/Volumes/BACKUP/venv/bin/python3` |
| `[RED] ... Command Line Tools ... no iOS SDK` | Xcode no está seleccionado (ver la primera fila) |
| Se acaba el espacio en disco durante el build | Detén el build (Ctrl+C), libera espacio y vuelve a lanzarlo con el mismo `--work-dir`: retoma lo ya hecho |

---

## Aviso

Esta es una guía no oficial escrita por la comunidad, sin relación con Nintendo ni con el proyecto nxnative. Usa solo juegos y llaves que hayas extraído legalmente de hardware propio.

## Licencia

Esta guía está bajo la licencia [Creative Commons Atribución 4.0 Internacional (CC BY 4.0)](LICENSE). Puedes compartirla y adaptarla, siempre que des el crédito correspondiente.
