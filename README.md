# git-verify

Lista los repositorios de **GitHub** con cambios pendientes bajo un árbol de
directorios. Pensado para responder a _"¿hay algo que me olvidé de pushear o
de pullear entre mis máquinas?"_.

Es un wrapper chiquito (un solo script bash) sobre
[`mgitstatus`](https://github.com/fboender/multi-git-status), una herramienta
probada con 500+ stars. `git-verify` agrega tres cosas que `mgitstatus` solo no
hace:

1. Detección recursiva automática (sin registrar repos manualmente).
2. Filtro por remoto en `github.com` (ignora GitLab, Bitbucket, locales, etc.).
3. `--fetch` opcional para que el aviso de _"needs pull"_ sea preciso.

**Read-only por default.** No hace `push`, no hace `pull`, no toca tu código.
Solo lee.

---

## Qué detecta

Para cada repo GitHub encontrado:

| Estado           | Significa                                 | Acción típica            |
| ---------------- | ----------------------------------------- | ------------------------ |
| `Uncommitted`    | Cambios sin stagear o sin commitear       | `git add` + commit       |
| `Untracked`      | Archivos nuevos sin trackear              | `git add` o `.gitignore` |
| `Needs push`     | Commits locales que no están en el remoto | `git push`               |
| `Needs pull`     | Commites en el remoto que no tenés local  | `git pull`               |
| `Needs upstream` | Branch sin remoto configurado             | `git push -u`            |
| `Stashes`        | Cambios guardados en stash                | `git stash pop`          |

---

## Instalación

### 1. Dependencias del sistema

```bash
# bash, git y find ya vienen en cualquier Linux/macOS. Falta mgitstatus:
git clone https://github.com/fboender/multi-git-status.git
cd multi-git-status
make install                                  # global (con sudo)
# o sin sudo, solo para tu usuario:
PREFIX=~/.local make install
```

### 2. git-verify

```bash
git clone <url-de-este-repo> ~/code/git-verify
ln -s ~/code/git-verify/bin/git-verify ~/.local/bin/git-verify
# asegurate de tener ~/.local/bin en tu PATH
```

---

## Uso

```bash
git-verify                              # escanea $HOME, depth 2, sin fetch
git-verify --fetch                     # igual pero hace 'git fetch' antes
git-verify --all                        # muestra también los repos sin cambios
git-verify ~/proyectos                  # escanea otra raíz
git-verify -d 4 --fetch ~/code          # profundidad 4 + fetch
git-verify -d 0 ~/proyectos             # sin límite de profundidad
git-verify --help                       # ayuda completa
```

### Flags

| Flag               | Default | Qué hace                                                                             |
| ------------------ | ------- | ------------------------------------------------------------------------------------ |
| `[DIRECTORIO]`     | `$HOME` | Raíz del escaneo.                                                                    |
| `-d N` / `--depth` | `2`     | Profundidad máxima. `0` = sin límite. Cuenta el dir de partida: `raíz/<x>/.git` = 2. |
| `--fetch`          | off     | Hace `git fetch --quiet` antes. Toca cada remote.                                    |
| `--all`            | off     | Muestra también los repos OK (sin cambios).                                          |
| `-h` / `--help`    | —       | Ayuda.                                                                               |

### Exit codes

`git-verify` hereda los exit codes de `mgitstatus`, que solo distingue éxito
de ejecución (no "hay cambios pendientes"):

- `0` — ejecución exitosa (puede haber o no haber cambios pendientes, mirar stdout).
- `1` — error interno de `mgitstatus` (caso raro).
- `2` — error de uso (argumentos inválidos, directorio inexistente).
- `127` — falta `mgitstatus`.

Si necesitás colgarlo de un cron/alerta que reaccione a "hay cambios", parseá
stdout o usá `grep -q ": ok" /dev/null` después. Una alternativa es
`git-verify --all | grep -v 'ok'` para ver solo los problemáticos.

---

## Por qué un wrapper y no un script propio desde cero

`mgitstatus` ya resuelve las decenas de casos borde de git (worktrees,
submodules, diverged, stashes, sin upstream…). Reinventarlo cuesta cientos de
líneas y aparecen bugs de estados mal interpretados. Este repo solo agrega la
lógica que `mgitstatus` no trae: detección recursiva automática + filtro por
GitHub + fetch opcional con output claro.

Si `mgitstatus` te queda grande, podés borrarlo del medio y quedarse con un
`git status` por repo en un loop. Pero entonces perdés toda la riqueza de
estados que muestra.

---

## Limitaciones conocidas

- **Performance en `$HOME`**: con `--depth 2`, escanea `$HOME` completa.
  Suele tardar segundos. Con muchos miles de subdirs podés verlo lento;
  pasale un DIRECTORIO más chico o bajale `--depth`.
- **Paralelización**: la versión actual es secuencial para mantener simple el
  código. Si te molesta la velocidad con muchos repos, abrí un issue/PR.
- **Remotes distintos de `origin`**: se chequean todos los remotes del repo,
  no solo `origin`. Si uno solo matchea `github.com`, el repo entra.
- **GitHub Enterprise** (`github.empresa.com`): entra como GitHub porque el
  filtro es substring. Si querés excluirlo, abrí un issue.

---

## Licencia

MIT (mismo espíritu que `mgitstatus`).
