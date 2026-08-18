---
name: adoptar-biblioteca
description: Instala y adopta la biblioteca de skills en un repositorio. Úsala cuando el proyecto va a empezar a usar el modelo de trabajo, cuando las skills de la biblioteca no aparecen o no dictaminan, cuando falta .claude/PERFIL-DEL-REPO.md, o cuando el repo todavía tiene copias locales de skills en .claude/skills/. También verifica que el plugin haya cargado de verdad.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
allowed-model-invocation: true
---

# Adoptar la biblioteca en este repositorio

Convierte un repo cualquiera en un repo que puede ser revisado por la biblioteca. Son cinco pasos
y ninguno es opcional: saltarse el 2 deja las skills cargadas pero mudas, y saltarse el 4 deja dos
definiciones contradictorias de cómo se revisa aquí.

> **La regla que ordena todo lo demás:** una skill de la biblioteca no menciona una sola ruta, un
> nombre de librería, un nombre de campo ni un nombre de app. Todo eso lo pide al perfil. Por eso
> **un repo sin perfil no es que revise mal: es que no revisa.** Cada punto sale `NO VERIFICABLE`.

## Antes de empezar: comprobar que el plugin cargó

No se supone. Se comprueba, y falla distinto según dónde estés:

| Dónde | Cómo se comprueba | Si no aparece |
|---|---|---|
| Claude Code (terminal o pestaña Code) | `/plugin` → **Installed**, y las skills se invocan como `/biblioteca-de-skills:<skill>` | Ver §Diagnóstico |
| Cowork | El menú `Customize → Plugins` lo lista como instalado | Cowork **no lee `.claude/settings.json`**. Se instala desde `Customize`, no desde el repo |
| Sesión en la nube | Se instala al arrancar la sesión desde `.claude/settings.json` | Falta `enabledPlugins`, o el marketplace no es alcanzable sin credenciales |

Si esta skill se está ejecutando, el plugin cargó. Anótalo y sigue.

---

## Paso 1 · El perfil del repo

Copia la plantilla y **rellénala entera**:

```bash
mkdir -p .claude
cp "${CLAUDE_PLUGIN_ROOT}/skills/adoptar-biblioteca/plantillas/PERFIL-DEL-REPO.md" .claude/PERFIL-DEL-REPO.md
```

Después, recorre el repo y propón un valor para cada clave, **citando `archivo:línea` de dónde lo
sacaste**. Reglas de llenado, en orden de importancia:

1. **Un valor propuesto no es un valor decidido.** Marca cada uno como `[propuesto]` hasta que la
   persona lo confirme. Un valor deducido del código se ve idéntico a uno decidido, y la siguiente
   revisión lo hereda como si alguien lo hubiera pensado.
2. **`ninguno` es un valor; la ausencia no lo es.** `frontend.estado_servidor: ninguno — axios +
   useState` habilita un `N/A` legítimo. La clave vacía detiene el punto.
3. **`backend.revocacion_de_sesion` no se rellena leyendo `settings.py`.** Es la clave que dio
   origen a la Regla 1 del protocolo: un repo puede tener `BLACKLIST_AFTER_ROTATION = True` y no
   tener revocación, porque la app que la implementa no está en `INSTALLED_APPS`. Se rellena
   habiendo cerrado sesión y reintentado con el refresh viejo.
4. **Nunca inventes una clave nueva.** La lista es cerrada: agregar una clave es un cambio del
   plugin, no una decisión local.

Deja a la vista, al final del archivo, la lista de claves que quedaron sin decidir y qué punto de
qué skill apaga cada una. Ese es el trabajo pendiente, y es media hora, no un proyecto.

## Paso 2 · La lista de despliegue

```bash
mkdir -p docs
cp "${CLAUDE_PLUGIN_ROOT}/skills/adoptar-biblioteca/plantillas/05-despliegue.md" docs/05-despliegue.md
```

Aquí viven los puntos que **no se pueden verificar desde el repo** — backups con restauración
probada, variables de entorno del panel, capacidades que dependen de configuración. Un punto de esta
lista sin fecha es un punto que nadie ha comprobado.

## Paso 3 · Desincrustar el `CLAUDE.md`

Quita del `CLAUDE.md` del repo **todo hecho que ahora viva en el perfil** y déjalo apuntando a él.

- `CLAUDE.md` se queda con la prosa: el porqué, las trampas heredadas, lo que está en producción y
  no se toca.
- El perfil se queda con los hechos.
- **Si un dato aparece en los dos y difieren, gana el perfil.** Escríbelo así, literal, en el
  `CLAUDE.md`.

Un dato en dos lugares es un dato que va a estar mal en uno de los dos. El plazo medido no fueron
tres meses: fueron tres horas.

## Paso 4 · Archivar las skills y los agentes locales

```bash
ls .claude/skills/ .claude/agents/ 2>/dev/null
```

Si hay algo, **no se borra: se archiva**, y no se deja conviviendo con el plugin.

```bash
mkdir -p .claude/_archivo
git mv .claude/skills .claude/_archivo/skills-locales   # si están versionadas
git mv .claude/agents .claude/_archivo/agentes-locales
```

Antes de archivar, **cosecha**: lee cada skill local y separa lo que es regla universal de lo que es
hecho local. Lo universal que la biblioteca no tenga es material para el camino de subida (§Camino
de subida). Lo local se muda al perfil. Lo demás se archiva y ya.

> Este paso no es orden: es corrección. Una skill local de un repo real llegó a mandar comprobar una
> bandera de configuración que estaba en `True` y no hacía nada, y con eso **aprobaba el P0 más
> grave del proyecto**. Dos definiciones de "cómo se revisa aquí" producen revisiones que se
> contradicen, y siempre gana la que es más fácil de contestar.

## Paso 5 · La línea base

Corre el agente `reviewer` en **modo auditoría** sobre todo el código y produce `docs/00-deuda.md`,
con cada hallazgo en `P0`/`P1`/`P2` y su `archivo:línea`.

**No bloquea nada.** El estándar estricto aplica desde `proyecto.fecha_adopcion` hacia adelante; lo
anterior es backlog priorizado. *Clean as you code*: sin esta regla el revisor devuelve
"BLOQUEA (34 puntos)" el primer día, el proyecto se detiene y el proceso se apaga a la semana. La
línea base solo puede mejorar.

Escribe la fecha de adopción en `proyecto.fecha_adopcion` **el día que corras este paso**, no antes.

---

## Cierre: la adopción terminó cuando

- [ ] `.claude/PERFIL-DEL-REPO.md` existe y ninguna clave está vacía (`ninguno` cuenta como llena)
- [ ] `docs/05-despliegue.md` existe
- [ ] `CLAUDE.md` no repite ningún valor del perfil y dice quién gana si difieren
- [ ] `.claude/skills/` y `.claude/agents/` locales están archivados o no existían
- [ ] `docs/00-deuda.md` está generado y `proyecto.fecha_adopcion` tiene fecha
- [ ] `MEMORIA.md` dice que este repo adoptó la biblioteca, y en qué fecha

Si el repo ya venía construido, esta skill cubre el paso previo de las cinco sesiones de adopción;
el contrato inverso y el triage van aparte.

---

## Diagnóstico: el plugin no aparece

En orden, del fallo más común al más raro:

1. **Estás en Cowork y lo configuraste en el repo.** Cowork toma sus skills, plugins y connectors de
   `Customize`, que sincroniza con la cuenta de claude.ai — **no del `.claude/` del repo ni de
   `~/.claude/`**. Son dos instalaciones distintas y hay que hacer las dos.
2. **No confiaste la carpeta.** En Claude Code el marketplace de `.claude/settings.json` se registra
   cuando se confía el directorio del proyecto. Sin eso no se lee.
3. **El formato de `enabledPlugins` es inválido y se ignora en silencio.** El valor válido es `true`:
   `"biblioteca-de-skills@biblioteca-emanuel": true`. Un objeto `{ "enabled": true }` **no da error,
   simplemente no hace nada.**
4. **El nombre del marketplace no coincide.** La clave de `extraKnownMarketplaces` tiene que ser
   idéntica al campo `name` del `marketplace.json` del repo del marketplace.
5. **El marketplace es privado y git no puede autenticarse.** Compruébalo directamente:
   ```bash
   git ls-remote https://github.com/<usuario>/<repo-del-marketplace>
   ```
   Si pide usuario y contraseña, el plugin nunca va a instalar. Para GitHub por HTTPS:
   `gh auth setup-git`.
6. **El marketplace está viejo en caché.** `/plugin marketplace update <nombre>`.

Y una advertencia sobre los repos privados: el refresco en segundo plano **desactiva los credential
helpers**, así que su `git pull` no se autentica por HTTPS, falla, borra el clon y vuelve a clonar.
Un marketplace privado por HTTPS falla de forma intermitente por diseño. Un repo público no tiene
ninguno de estos seis problemas.

---

## Camino de subida

Cuando este proyecto corrija, afine o agregue un punto a una skill, **el cambio se hace en el
plugin, no en la copia local — y se publica en el mismo movimiento.** Un punto nacido de un
incidente que no llega al plugin está garantizado que se repite en el siguiente proyecto.

Y antes de agregarlo, las dos reglas de admisión:

1. **Ningún punto entra si se contesta abriendo un archivo y mirando un valor.** Declara el efecto
   observable y cómo provocarlo, o se marca `NO VERIFICABLE` y se muda a la lista de despliegue.
2. **Solo entra lo que ya se ejercitó en un proyecto real.** Si no puedes nombrar el proyecto y el
   día en que ese problema ocurrió, todavía no es un punto.

Si una sesión tocó una skill, **`MEMORIA.md` lo dice.** Es el detector: sin él, tres skills nacieron
dentro de un repo y nunca subieron.
