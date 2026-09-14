# git-verify

Escanea un árbol de directorios y lista los repos git que encuentra,
**separando los que apuntan a GitHub de los que apuntan a otros remotes**.
Para cada repo muestra **info útil** (branch, remote, estado de cambios).

Pensado para responder a _"¿qué tengo pendiente entre mis máquinas / entre
push y pull, en todos mis proyectos?"_ sin tener que entrar a cada uno a hacer
`git status` a mano.

Sin dependencias obligatorias más allá de **bash, git y find**. `mgitstatus`
solo se necesita si usás `--detailed`.

---

## Ejemplo de salida

```text
$ git-verify --fetch --only-pending

git-verify  7 repos · 3 con cambios pendientes
fetch activado · 4 jobs en paralelo
filtro: solo pendientes

▶ GitHub (7 repos · 3 con cambios)
------------------------------------------------------------
  ! carta-qr  1 behind
  ! kioscoGustavo  44 behind
  ! tuAmigoFI-viejo  34 unstaged · 8 untracked · 98 behind

▶ Otros remotes (0 repos · 0 con cambios)
------------------------------------------------------------
  (nada en esta categoría)

fetch: 7 ok · 0 fallaron (sin auth o sin red)
```

Sin `--only-pending`, los repos clean también se listan (en su propio bloque
`─ clean (N) ─`) y el remote tag se muestra solo en la sección **Otros remotes**
(la sección GitHub lo omite para no ser redundante). El ícono y color solo
aparecen si stdout es una terminal interactiva.

`✓` indica árbol de trabajo limpio: sin staged/unstaged/untracked, sin commits
locales sin pushear (ahead) y sin commits del remoto sin pullear (behind).
`!` indica cualquier desviación: cambios locales sin commitear, archivos sin
trackear, commits ahead (sin pushear), commits behind (sin pullear), o falta
de upstream configurado.

---

## Instalación

### Dependencias

Solo **bash, git y find** vienen en cualquier Linux/macOS.

`mgitstatus` (opcional, solo con `--detailed`):

```bash
git clone https://github.com/fboender/multi-git-status.git
cd multi-git-status
make install                                  # global (con sudo)
# o sin sudo, solo para tu usuario:
PREFIX=~/.local make install
```

### git-verify

```bash
git clone <url-de-este-repo> ~/code/git-verify
ln -s ~/code/git-verify/bin/git-verify ~/.local/bin/git-verify
# asegurate de tener ~/.local/bin en tu PATH
```

---

## Uso

```bash
git-verify                              # escanea $HOME entero, sin límite, sin fetch
git-verify --only-pending               # oculta los repos "clean"
git-verify --fetch                      # hace 'git fetch' antes (necesario para "needs pull" preciso)
                                           # con 4 fetches en paralelo por default
git-verify --fetch --parallel 8         # 8 fetches simultáneos (max 16)
git-verify --fetch --no-parallel        # forzar serial (un fetch por vez)
git-verify --detailed                   # agrega sección detallada de los GH (usa mgitstatus)
git-verify ~/proyectos                  # escanea otra raíz
git-verify -d 3 ~/code                  # limita a 3 niveles
git-verify --help                       # ayuda completa
```

### Flags

| Flag               | Default | Qué hace                                                                                       |
| ------------------ | ------- | ---------------------------------------------------------------------------------------------- |
| `[DIRECTORIO]`     | `$HOME` | Raíz del escaneo. Solo se usa el primer posicional.                                            |
| `-d N` / `--depth` | `0`     | Profundidad máxima. `0` = sin límite (default). Cuenta el dir de partida: `raíz/<x>/.git` = 2. |
| `--fetch`          | off     | Hace `git fetch --quiet` antes. Toca cada remote.                                              |
| `--parallel N`     | `4`     | Fetches simultáneos (solo con `--fetch`). Default 4, máximo 16. Más no ayuda (red saturada).   |
| `--no-parallel`    | off     | Desactiva paralelismo, fuerza serial con `--fetch`.                                            |
| `--only-pending`   | off     | Oculta los repos clean (cambia el default de "mostrar todos" a "solo problemáticos").          |
| `--detailed`       | off     | Al final, llama a `mgitstatus` para info detallada de los GH (necesita mgitstatus).            |
| `--include-all`    | off     | No excluye las carpetas de "ruido" (ver abajo). Por default se filtran.                        |
| `-h` / `--help`    | —       | Ayuda.                                                                                         |

Comportamiento automático:

- **Color** (`!` rojo, `✓` verde) se aplica **solo si stdout es TTY**. Si lo
  pipeás a un archivo o a otro comando, sale sin color.
- **`clear` al inicio** se aplica **solo si stdout es TTY**. Pipeado a `| tee`
  o `> archivo`, no limpia.
- **`GIT_TERMINAL_PROMPT=0`** se setea automáticamente, así que `--fetch` no
  queda colgado pidiendo credenciales. Si algún repo falla por auth, se
  loguea al final: `fetch: 28 ok · 2 fallaron`.

### Qué significa cada ícono

| Ícono | Significa                                                                                                            |
| ----- | -------------------------------------------------------------------------------------------------------------------- |
| `✓`   | Working tree limpio: sin cambios locales, sin commits ahead ni behind.                                               |
| `!`   | Hay cambios: staged, unstaged, untracked, commits ahead (sin pushear), commits behind (sin pullear), o sin upstream. |
| `-`   | Repo bare (sin working dir).                                                                                         |

---

## Por qué dos secciones (GitHub / Otros)

Tu pedido original fue filtrar por GitHub. Pero cuando escaneás `$HOME` te
vas a encontrar con un montón de clones auxiliares (versiones de node, plugins,
experimentos) que apuntan a GitHub sin ser "tus proyectos". **Otros remotes**
te deja ver de un vistazo qué hay fuera de tu flujo principal de GitHub, sin
perder visibilidad.

Si te molesta verlos, pasale un directorio más específico en vez de `$HOME`.

---

## Carpetas excluidas por default

Para que el output no se contamine con ruido que no son proyectos tuyos,
por default se **excluyen** estas carpetas bajo la raíz del escaneo:

| Path                   | Qué es                                         |
| ---------------------- | ---------------------------------------------- |
| `~/.cache`             | Caches de paquetes (paru/AUR, pip, npm, etc.)  |
| `~/.local/share/Trash` | Papelera de Linux                              |
| `~/.nvm`               | Versiones de Node instaladas (nvm)             |
| `~/.pi/agent`          | Runtime de Pi (herramientas instaladas por Pi) |

Para ver TODO (incluyendo esas carpetas), usá `--include-all`.

Si querés agregar más exclusiones, abrí un issue o modificá la variable
`NOISE_DIRS` al principio del script.

---

## Comparación con `mgitstatus` solo

`mgitstatus` es excelente y más detallado (sabe de worktrees, submodules,
diverged, stash...), pero:

| Cosa                      | `mgitstatus` solo  | `git-verify`                                            |
| ------------------------- | ------------------ | ------------------------------------------------------- |
| Detección automática      | Sí (con `-r`)      | Sí (siempre)                                            |
| Filtro GitHub             | **No**             | **Sí**                                                  |
| Diferencia GH / no-GH     | No (mezcla todo)   | Sí, en secciones separadas                              |
| Branch + remote por repo  | No                 | Sí (compacto, branch solo si no es estándar)            |
| Output summary con conteo | No                 | Sí ("30 repos · 14 con cambios")                        |
| Detalle fino (stash etc.) | Sí                 | Sí, con `--detailed`                                    |
| Fetch tolera fallos auth  | **No** (se cuelga) | **Sí** (GIT_TERMINAL_PROMPT=0, loguea los que fallaron) |
| Clear / color en TTY      | No                 | Sí (TTY-aware, no rompe pipes)                          |
| Exit code "hay cambios"   | **No** (siempre 0) | Tampoco (parseá stdout si lo necesitás en scripts)      |

`git-verify` es lectura + info clara. Para actuar, vas al repo y hacés lo tuyo.

---

## Limitaciones conocidas

- **Performance**: con el default (`--depth 0`) escanea `$HOME` entera. Suele
  tardar pocos segundos. Con home con muchos miles de subdirs, bajale el
  `--depth` o pasale un dir más chico.
- **Submodules**: el script entra solo a `.git` directorios, así que no entra
  recursivamente en submodules. Suficiente para escaneo rápido; si necesitás
  info de submodules, andá directo con `git submodule status`.
- **Repos bare** (sin working dir): se marcan con `-` y `[bare]` porque no
  tienen branch/remote de la forma usual. No se les aplica fetch.
- **Repos con múltiples remotes**: se considera "GitHub" si **alguno** matchea
  `github.com`. El remote que se muestra es `origin` o el primero disponible.
- **GitHub Enterprise** (`github.empresa.com`): entra como GitHub porque el
  filtro es substring. Si querés excluirlo, abrí un issue.
- **Exit code**: `git-verify` hereda el comportamiento de `mgitstatus` y
  siempre devuelve `0` mientras ejecute bien. Si necesitás alertar en base a
  "hay cambios pendientes", parseá stdout (el script marca con `!` los
  problemáticos, así que `git-verify --only-pending | grep -q '^  !'` sirve).

---

## Licencia

MIT.
