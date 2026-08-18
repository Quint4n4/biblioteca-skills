---
name: protocolo-de-revision
description: >
  La cabecera común de la biblioteca: cómo se emite un veredicto, qué cuenta como evidencia, cómo se
  lee el perfil del repo y qué forma tiene un punto admisible. Úsala SIEMPRE que se revise, audite o
  cierre cualquier cosa, y cárgala ANTES que cualquier otra skill de la biblioteca — las demás
  suponen que sus reglas ya están puestas y no las repiten. Define los cuatro veredictos (BLOQUEA,
  PASA, N/A, NO VERIFICABLE), la regla de que sin archivo:línea no hay veredicto, la escala P0/P1/P2,
  los modos gate y auditoría, y el camino de subida de un punto nuevo a la biblioteca.
---

# Protocolo de revisión

Esta skill no revisa nada. Define **cómo se revisa**, una sola vez, para que las otras seis no
tengan que repetirlo y no puedan contradecirlo.

Si estás cargando cualquier skill de esta biblioteca y no cargaste esta, detente y cárgala. Las
demás están escritas suponiendo que estas reglas ya están puestas.

---

## 1 · Los cuatro veredictos

Cada punto de cada checklist se responde con exactamente uno:

| Veredicto | Qué significa | Qué exige |
|---|---|---|
| **BLOQUEA** | Comprobé el efecto y ocurrió lo que no debía | `archivo:línea` + qué pasaría en producción |
| **PASA** | Comprobé el efecto y ocurrió lo que debía | `archivo:línea` de dónde está resuelto |
| **N/A** | El punto no aplica a este repo | La **clave del perfil** que lo declara inaplicable |
| **NO VERIFICABLE** | No se puede comprobar desde el repo | Qué haría falta + destino en `docs/05-despliegue.md` |

### La regla de evidencia

> **Sin `archivo:línea` no hay veredicto. La ausencia de evidencia nunca es PASA.**

Los tres corolarios que la hacen funcionar, y que son los que se rompen en la práctica:

- **`PASA` también necesita cita.** "No encontré el problema" y "verifiqué que está resuelto aquí"
  no son lo mismo. El primero es `NO VERIFICABLE`.
- **`N/A` no se justifica con una impresión.** Se justifica citando la clave del perfil que lo
  declara: `aislamiento.ambito: ninguno`. Sin esa clave, es `NO VERIFICABLE`, no `N/A`.
- **`NO VERIFICABLE` no es una nota al pie.** Es un punto que sale de la revisión y **entra a
  `docs/05-despliegue.md`**. Si nadie lo mueve, desaparece — y desaparecer es exactamente lo que
  hace que un checklist dé aprobaciones falsas.

### Un solo BLOQUEA detiene el módulo

No existe "bloquea pero es menor". Si de verdad fuera menor, no sería un punto del checklist y
sobra del checklist.

---

## 2 · Las dos reglas de admisión

Estas dos reglas deciden **qué puede ser un punto**. Un punto que no las cumple no se suaviza: se
saca.

### Regla 1 — Ningún punto entra si se contesta abriendo un archivo y mirando un valor

Cada punto declara **el efecto observable y cómo provocarlo**.

```
❌  Revisar que ROTATE_REFRESH_TOKENS = True y BLACKLIST_AFTER_ROTATION = True

✅  Cerrar sesión y reintentar con el refresh viejo → debe responder 401.
    Si responde 200, la revocación no existe: BLOQUEA.
```

**Por qué es la regla más importante de la biblioteca:** el punto fácil siempre gana sobre el
difícil. Un auditor con prisa contesta el que se lee y salta el que se comprueba, y los dos suman
igual en el reporte. Ese es el mecanismo exacto por el que una skill de esta biblioteca llegó a
aprobar un P0 real: pedía comprobar que una bandera estuviera en `True`, y estaba en `True`, y no
servía de nada porque la aplicación que la hace funcionar no estaba instalada.

Leer configuración es válido **solo como parte de un punto cuyo efecto también se provoca**. Nunca
como el punto entero.

Lo que no se pueda expresar así se marca `NO VERIFICABLE DESDE EL REPO` y se muda a la lista de
despliegue. **No se conserva por lástima.** Un punto que no se puede fallar solo reparte
aprobaciones.

### Regla 2 — Solo entra lo que ya se ejercitó en un proyecto real

Nada escrito por si acaso. Una skill escrita y nunca ejercitada es una suposición larga.

Corolario práctico: los mejores puntos de esta biblioteca nacieron en una auditoría o en un
incidente, no en una sesión de diseño. Si estás por agregar un punto y no puedes nombrar el
proyecto y el día en que ese problema ocurrió, todavía no es un punto.

### Forma canónica de un punto

Toda tabla de revisión de esta biblioteca tiene estas cuatro columnas, en este orden:

| # | Punto | Cómo se comprueba | BLOQUEA si |
|---|---|---|---|
| 1 | La revocación de sesión existe | Cerrar sesión, reintentar con el refresh viejo | Responde 200 |

La columna **Cómo se comprueba** contiene una acción o un comando, no una descripción. Si al
escribirla te sale "revisar que…", el punto no cumple la Regla 1.

---

## 3 · El perfil del repo

Todo lo local vive en **`.claude/PERFIL-DEL-REPO.md`**, con una **lista de claves cerrada**
documentada en la plantilla de la biblioteca. Ninguna skill de esta biblioteca menciona una ruta,
un nombre de librería, un nombre de campo ni un nombre de app: los pide al perfil.

### Las tres reglas del perfil

**1 · Clave ausente = `NO VERIFICABLE` y alto.**

> Si una skill necesita una clave que el perfil no trae, el punto se marca `NO VERIFICABLE`,
> se anota qué clave faltó, y **no se sigue con ese punto**.
>
> **Nunca se adivina. Nunca se infiere del código.** Un valor deducido leyendo el repo se ve igual
> que uno declarado y no lo es: es una suposición con formato de hecho, y la siguiente revisión la
> hereda como si alguien la hubiera decidido.

**2 · `ninguno` es un valor; la ausencia no lo es.**

`estado_servidor: ninguno — axios + useState` y una clave `estado_servidor` que no aparece son
cosas distintas y producen veredictos distintos. La primera declara una decisión y habilita un
`N/A` legítimo. La segunda es un hueco y produce `NO VERIFICABLE`.

**3 · El perfil se queda con los hechos; `CLAUDE.md` con la prosa.**

> El `CLAUDE.md` del repo **apunta al perfil y no repite ni un valor.** Si un dato aparece en los
> dos y difieren, **gana el perfil.**

`CLAUDE.md` conserva lo que el perfil no puede tener: el porqué de una decisión, las trampas
heredadas, las convenciones a obedecer, lo que está en producción y no se toca. El perfil conserva
los hechos que las skills consultan. Un dato en dos lugares es un dato que en tres meses está mal
en uno de los dos.

---

## 4 · Excepciones locales

Una desviación **documentada** pasa. Una desviación **no documentada** bloquea.

La excepción vive en el perfil, en su sección de excepciones, y lleva tres cosas: qué regla se
desvía, por qué, y qué la volvería a hacer aplicable.

Ejemplo real: el interceptor de 401 de una PWA que no refresca el token, limpia la sesión y manda a
login, mientras el panel administrativo del mismo repo sí refresca. Es distinto, es correcto, y sin
declararlo un auditor lo reporta como bug cada vez que pasa por ahí.

---

## 5 · Los dos modos

### Modo gate (por defecto)

Se revisa un módulo recién escrito, antes de acercarse a `main`. Un solo BLOQUEA lo regresa a
construcción.

### Modo auditoría

Se pide explícitamente, al adoptar el proceso en un proyecto que ya existe. Se barre todo el código
heredado y **aquí no se bloquea nada**: se produce `docs/00-deuda.md`, un backlog clasificado.

**Por qué existe este modo:** aplicar la regla de gate a código que ya existe produce un veredicto
de "BLOQUEA (34 puntos)" que paraliza el proyecto, y un proceso que paraliza se apaga a la semana.
La estrategia se llama *clean as you code*: el estándar estricto aplica al código escrito **a partir
de la fecha de adopción**, que el perfil declara en `proyecto.fecha_adopcion`. Lo viejo entra por
backlog. La línea base solo puede mejorar, nunca empeorar.

### Escala de severidad (la misma en los dos modos)

| Nivel | Qué es | Cuándo se arregla |
|---|---|---|
| **P0** | Puede exponer datos de un ámbito a otro, permitir acceso no autorizado, o perder datos de forma irreversible | Antes del siguiente despliegue a producción, o del alta del siguiente ámbito |
| **P1** | Degrada el servicio o impide detectar fallas: N+1, sin monitoreo de errores, sin backup probado, sin bitácora | En las siguientes semanas |
| **P2** | Deuda técnica sin consecuencia inmediata | Cuando se toque ese archivo |

**La severidad se justifica con un escenario, no con un adjetivo.** "Grave" no es una severidad;
"la recepcionista de la sucursal Norte ve el teléfono de un paciente de Centro" sí lo es. El
escenario dice qué rol, en qué pantalla, con qué dato.

---

## 6 · Entregables

| Modo | Archivo | Cierra con |
|---|---|---|
| Gate | `docs/04-revision-<modulo>.md` | `VEREDICTO: BLOQUEA (N puntos)` o `VEREDICTO: PASA` |
| Auditoría | `docs/00-deuda.md` | Hallazgos ordenados por gravedad, no por orden de aparición |
| Cualquiera | `docs/05-despliegue.md` | Se **añaden** los `NO VERIFICABLE` encontrados |

Si `BLOQUEA`, cada punto lleva una línea con **qué pasaría en producción** si se ignora. Quien
decide lo hace con consecuencias concretas, no con categorías abstractas de severidad.

**Con `verificadores.ci: ninguno`, el reporte lo dice en su primera línea.** Sin integración
continua, ningún verificador corre solo: todos los `PASA` de esta revisión valen para el commit que
se revisó y para ninguno posterior. No es un `BLOQUEA` —es una propiedad del repo, no del módulo—
pero omitirlo hace que un reporte de hace tres semanas se lea como si siguiera vigente.

**Contradecir el contrato es BLOQUEA automático.** Si el código contradice `proyecto.contrato`, o
el código está mal o el contrato quedó desactualizado. Las dos cosas se arreglan antes de fusionar:
un contrato desactualizado hace que el siguiente agente construya sobre una mentira.

### Lo que no va en una revisión

No propongas refactors de estilo, opiniones sobre nombres, ni nada de diseño visual. Solo el
checklist. El ruido hace que se dejen de leer las revisiones, y una revisión que no se lee no
protege nada.

---

## 7 · El camino de subida

Es lo único de este documento que evita que la biblioteca vuelva a divergir.

> **Cuando un proyecto corrige, afina o agrega un punto a una skill, el cambio se hace en el
> plugin, no en la copia local — y se publica en el mismo movimiento.**

Un arreglo no termina en el commit. Termina cuando la causa se agrega como punto nuevo en la skill
correspondiente, para que la próxima revisión lo atrape. Un punto nacido de un incidente que no
llega al plugin está garantizado que se repite en el siguiente proyecto.

Hasta ahora había camino de bajada y no de subida. No se omitió por descuido: **no había regla que
lo pidiera.** Esta es esa regla, y le falta la mitad que siempre falta — **en qué skill, y dónde
vive esa skill**:

1. El punto se escribe cumpliendo las dos reglas de admisión de §2.
2. Se agrega al `SKILL.md` del plugin, no al `.claude/skills/` del repo.
3. Se publica; los repos lo reciben con `/plugin marketplace update`.
4. Si la sesión tocó una skill, **`MEMORIA.md` lo dice**. Es lo que habría delatado en su momento
   que tres skills nacieron dentro de un repo y nunca subieron.

### Entre dos definiciones de la misma regla, gana la que trae el verificador

Cuando dos skills, o una skill y un `CLAUDE.md`, definen el mismo requisito de formas distintas:
manda la que trae el test o el comando que lo comprueba. Las otras borran su versión corta y
apuntan a ella.

**Una regla sin verificador es una intención.**
