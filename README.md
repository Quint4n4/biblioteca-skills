# Biblioteca de skills — marketplace privado

Las siete skills de revisión y los cuatro agentes del modelo de trabajo, en un solo lugar del que
todos los proyectos tiran.

**Esto no es una reorganización de carpetas.** Es el arreglo de un error de diseño que ya produjo
una skill que aprobaba un P0: cada skill mezclaba en el mismo párrafo *la regla* —que vale en
cualquier proyecto— con *el hecho local* —que vale en un repo y es falso en el siguiente. Cuando las
dos van juntas, la skill entera se vuelve inutilizable fuera de su proyecto de origen, y el proyecto
siguiente escribe la suya, que dice casi lo mismo y diverge desde el primer día.

---

## Instalar

**Son dos instalaciones distintas y hay que hacer las dos.** Cowork y Claude Code no comparten
configuración de plugins: el `.claude/settings.json` de un repo **no lo lee Cowork**, y lo que se
instala en `Customize` no llega a la terminal. Confundirlos es el motivo más común de "instalé el
plugin y no aparece".

| Dónde trabajas | De dónde saca los plugins | Alcance |
|---|---|---|
| **Cowork** (app de escritorio) | `Customize → Plugins`, sincronizado con la cuenta de claude.ai | Todas las sesiones de Cowork, en cualquier carpeta |
| **Claude Code** (terminal y pestaña Code) | `.claude/settings.json` del repo, al confiar la carpeta | Solo ese repo |
| **Sesiones en la nube** | el mismo `.claude/settings.json`, al arrancar la sesión | Solo ese repo |

### 1 · Cowork — una sola vez

`Customize` en la barra lateral → pestaña **Plugins** → **Add from a repository** → la URL de este
repositorio. Sirve para todas las sesiones de Cowork; no hay que repetirlo por proyecto.

### 2 · Claude Code — una vez por repo

En el `.claude/settings.json` del repo (**sí se commitea**):

```json
{
  "extraKnownMarketplaces": {
    "biblioteca-emanuel": {
      "source": {
        "source": "github",
        "repo": "<usuario>/<repo-de-esta-biblioteca>"
      }
    }
  },
  "enabledPlugins": {
    "biblioteca-de-skills@biblioteca-emanuel": true
  }
}
```

Tres cosas que fallan **en silencio** si se escriben mal:

- El valor de `enabledPlugins` es `true`, un booleano. Un objeto `{ "enabled": true }` no da error:
  simplemente no hace nada.
- La clave de `extraKnownMarketplaces` tiene que ser **idéntica** al campo `name` de
  `.claude-plugin/marketplace.json`. Aquí es `biblioteca-emanuel`.
- El marketplace se registra **cuando se confía la carpeta del proyecto**. Sin eso no se lee.

Actualizar: `/plugin marketplace update biblioteca-emanuel`.

### 3 · Adoptar el repo — una vez por repo, después de instalar

```
/biblioteca-de-skills:adoptar-biblioteca
```

La skill lleva las plantillas dentro y hace los cinco pasos: perfil del repo, lista de despliegue,
desincrustar el `CLAUDE.md`, archivar las skills locales y generar la línea base de deuda.

**No es opcional.** Una skill de esta biblioteca no menciona una sola ruta, un nombre de librería ni
un nombre de campo: todo eso lo pide al perfil. Un repo instalado pero sin
`.claude/PERFIL-DEL-REPO.md` no revisa mal — **no revisa**: cada punto sale `NO VERIFICABLE`.

### Sobre repos privados

Este marketplace funciona mejor **público**. En privado, el refresco en segundo plano desactiva los
credential helpers de git, así que su `git pull` no se autentica por HTTPS, falla, borra el clon y
vuelve a clonar: los auto-updates fallan de forma intermitente por diseño. Si tiene que ser privado,
hay que correr `gh auth setup-git`, poner
`CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE=1` y actualizar a mano con
`/plugin marketplace update`. Nada de esto hace falta si el repo es público, y **el contenido de
estas skills es genérico: no lleva nada de ningún cliente.** Mantenerlo así es lo que permite que
siga siendo público.

**No se pueden habilitar skills sueltas de un plugin.** Se instala completo o se desactiva completo.
Y no importa: una skill bien escrita que no aplica **se declara `N/A` sola**, citando la clave del
perfil que la vuelve inaplicable. Por eso tenerlas todas en todos los proyectos no cuesta nada. Lo
que cuesta caro es una skill que **afirma** algo falso.

---

## Las tres capas

| Capa | Dónde vive | Qué contiene |
|---|---|---|
| **La regla** | `plugins/biblioteca-de-skills/skills/` | Universal. Sin rutas, sin nombres de librería, sin nombres de campo |
| **El hecho local** | `.claude/PERFIL-DEL-REPO.md` de cada repo | Lista de claves **cerrada**. `ninguno` es un valor; la ausencia no lo es |
| **La excepción** | Sección 7 del mismo perfil | Una desviación documentada pasa; una no documentada bloquea |

> Si una frase de una skill deja de ser cierta al cambiar de repo, no pertenece a la skill:
> pertenece al perfil.

---

## Qué trae

### Skills

| Skill | Para qué |
|---|---|
| `protocolo-de-revision` | **La cabecera. Se carga siempre y primero.** Los cuatro veredictos, la regla de evidencia, las dos reglas de admisión, la escala P0/P1/P2 y el camino de subida |
| `aislamiento-de-datos` | Que los datos de un ámbito no lleguen a otro, sea tenant o sede |
| `security-checklist` | Autenticación, revocación de sesión, permisos, entrada, secretos, datos personales |
| `django-backend` | Apps por dominio, servicios y selectores, atomicidad y concurrencia |
| `db-schema` | Tipos, borrado, índices, N+1, migraciones por pasos |
| `react-frontend` | Cliente HTTP central, tokens, 401/403/404, paginación |
| `auditoria-frontend` | Los cinco cruces contra el contrato que no se ven leyendo el backend |
| `adoptar-biblioteca` | Instala y adopta la biblioteca en un repo, y diagnostica por qué no cargó |

### Agentes

`architect` · `backend` · `frontend` · `reviewer`

El `reviewer` **no tiene herramientas de escritura**, a propósito: quien arregla no puede ser quien
dictamina, porque termina aprobando su propio parche.

---

## Estructura del repositorio

```
.claude-plugin/marketplace.json          catálogo del marketplace
plugins/biblioteca-de-skills/
  .claude-plugin/plugin.json             identidad del plugin
  skills/<nombre>/SKILL.md               una carpeta por skill
  skills/adoptar-biblioteca/plantillas/  PERFIL-DEL-REPO.md y 05-despliegue.md
  agents/<nombre>.md                     los cuatro agentes
```

Un plugin solo carga `skills/`, `commands/`, `agents/`, `hooks/`, `.mcp.json`, `.lsp.json`, `bin/` y
`settings.json`. **Por eso las plantillas viven dentro de la skill que las usa y no en una carpeta
suelta:** los archivos de apoyo de una skill sí viajan con el plugin, y la skill los alcanza con
`${CLAUDE_PLUGIN_ROOT}`. Una carpeta `plantillas/` en la raíz del plugin se sube a git y no llega a
ninguna sesión.

---

## La regla que evita que esto vuelva a divergir

Es lo único de este repositorio que de verdad importa. Sin esto, la biblioteca perfecta de hoy está
tres skills atrás en dos meses — que es exactamente lo que ya pasó una vez.

> **Cuando un proyecto corrige, afina o agrega un punto a una skill, el cambio se hace aquí, no en
> la copia local del repo — y se publica en el mismo movimiento.**

Hasta ahora había camino de bajada y no de subida. No se omitió por descuido: **no había regla que
lo pidiera.** Tres skills —las mejores del conjunto— nacieron dentro de un repo y nunca subieron.

Y el detector: **si una sesión tocó una skill, `MEMORIA.md` lo dice.** Es lo que habría delatado en
su momento que esas tres se quedaron abajo.

### Antes de agregar un punto, las dos reglas de admisión

1. **Ningún punto entra si se contesta abriendo un archivo y mirando un valor.** Cada punto declara
   el efecto observable y cómo provocarlo. Lo que no se pueda expresar así se marca
   `NO VERIFICABLE` y se muda a la lista de despliegue. **No se conserva por lástima.**
2. **Solo entra lo que ya se ejercitó en un proyecto real.** Si no puedes nombrar el proyecto y el
   día en que ese problema ocurrió, todavía no es un punto.

Y el criterio de desempate: **entre dos definiciones de la misma regla, gana la que trae el
verificador.** Una regla sin verificador es una intención.
