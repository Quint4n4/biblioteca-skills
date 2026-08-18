---
name: auditoria-frontend
description: >
  Checklist falsable para auditar un frontend contra el contrato del backend. Detecta las cinco
  clases de falla que no se ven leyendo solo el backend: deriva de permisos (botón fantasma y
  función invisible), gating por módulo desalineado, caché que sobrevive al cambio de ámbito o de
  sesión, transporte del ámbito ausente, y fugas del bundle. Úsala al auditar, al revisar en modo
  gate cualquier módulo que toque el frontend, y en cualquier sesión donde se pregunte si el
  frontend y el backend están alineados. Cada punto se responde con evidencia de AMBOS lados o se
  marca NO VERIFICABLE. No opina de diseño. Requiere `protocolo-de-revision`.
---

# Auditoría de frontend contra el contrato

Carga antes `protocolo-de-revision`. Esta skill supone sus reglas puestas y no las repite.

**No es una revisión de código ni de diseño: es un cruce de dos listas.** Cada punto se contesta
con evidencia `archivo:línea` de los dos lados, o se marca `NO VERIFICABLE`. Nunca `PASA` con una
sola cita.

## Claves del perfil que consume

`proyecto.contrato` · `proyecto.raiz_frontend` · `proyecto.raiz_backend` · `aislamiento.ambito` ·
`frontend.apps` · `frontend.estado_servidor` · `frontend.cliente_http` · `frontend.almacen_token` ·
`frontend.cache_offline` · `frontend.libreria_http` · `frontend.transporte_del_ambito` ·
`frontend.matriz_de_permisos` · `frontend.espejo_de_modulos`

---

## Por qué existe

El contrato documenta el backend y el backend es la autoridad de permisos. Eso hace creer que el
frontend no puede causar daño. Puede, de cinco formas, y **ninguna de las cinco se ve leyendo el
backend.**

---

## Cruce 1 · Deriva de permisos

Por cada permiso del backend, encuentra qué decide el frontend para esa misma acción. Dos fallas, y
las dos cuentan:

- **BOTÓN FANTASMA** — el frontend ofrece la acción y el backend responde 403. El usuario cree que
  el sistema está roto y llama. Es el más caro en soporte.
- **FUNCIÓN INVISIBLE** — el frontend la esconde y el backend la permite. Pagaste por código que
  nadie usa, y a veces es una capacidad que el cliente pidió y cree que no existe.

Entrega una tabla: `acción | roles que permite el backend (archivo:línea) | roles a los que el
frontend la muestra (archivo:línea) | veredicto`.

**Sin `frontend.matriz_de_permisos`, este cruce entero es `NO VERIFICABLE`.** No se reconstruye la
matriz leyendo los componentes: eso produce una matriz inventada que la siguiente auditoría hereda
como si alguien la hubiera decidido.

### Las tres trampas de este cruce

1. **El rol cacheado.** Si el rol se guarda al iniciar sesión, una comprobación hecha contra el rol
   viejo después de cambiar de usuario es una falla real, no teórica.
2. **Las reglas que no viven en el permiso.** Varias reglas del backend viven en el *service*, no en
   la clase de permiso: *el médico solo agenda para sí mismo*, *solo anula su propia receta*. Una
   matriz de rol no puede reflejarlas. Revisa si el frontend las esconde **por rol** cuando debería
   hacerlo **por pertenencia**.
3. **El código de error inesperado.** Un método no declarado puede responder 403 en vez de 405, y a
   veces depende del rol. Un manejo de error que espera 405 no se dispara nunca.

---

## Cruce 2 · Gating por módulo

`N/A` si `frontend.espejo_de_modulos: ninguno`.

Cada ruta protegida por módulo contra el requisito de las vistas que esa pantalla consume: deben
nombrar el mismo módulo.

- **Un módulo apagado responde 404, no 403** — el cliente no debe saber que existe algo que no
  compró. Si el frontend trata ese 404 como "no existe el registro" en vez de "no contratado", el
  usuario ve una pantalla vacía sin explicación.
- `frontend.espejo_de_modulos` es un **espejo** del catálogo del backend. Dos fuentes de verdad:
  compara los módulos, sus dependencias duras y los roles que cada uno habilita. Una divergencia
  hace que el menú ofrezca algo que la API va a negar.
- Verifica también el camino inverso: un módulo activo cuya pantalla no está enrutada.

---

## Cruce 3 · Caché entre ámbitos y entre sesiones

**El más grave y el que nadie revisa.** Es una fuga de datos que ocurre en el navegador aunque el
backend esté perfecto.

Dónde buscar depende de `frontend.estado_servidor`:

| Valor | Dónde vive el dato que puede sobrevivir |
|---|---|
| Una librería de caché | Sus claves de consulta |
| `ninguno` | El estado de los componentes, y sobre todo `frontend.cache_offline` |

**Con `estado_servidor: ninguno` este cruce no se apaga: se muda.** Una cola de sincronización
offline es exactamente donde el dato de un ámbito sobrevive a un cambio de sesión, y encima
sobrevive a cerrar el navegador.

Las cuatro preguntas:

1. ¿Toda clave de caché incluye el ámbito activo? Una clave como `['pacientes', filtros]` sirve
   datos del ámbito anterior al cambiar.
2. Al **cambiar de ámbito** con el selector: ¿qué se invalida? Lo que no se invalide sigue
   mostrando el anterior.
3. Al **cerrar sesión o cambiar de usuario**: ¿se vacía el caché de datos, no solo el token?
4. ¿Hay algo del dominio en almacenamiento del navegador? Revisa que no lleve datos personales y
   que esté separado por usuario y por ámbito.

Una clave sin ámbito es **P0**. Escribe el escenario exacto: qué usuario, qué pantalla, qué dato de
qué ámbito se ve.

---

## Cruce 4 · Transporte del ámbito

`frontend.transporte_del_ambito` dice cómo viaja el ámbito activo. Tres casos y tres revisiones
distintas:

| Valor | Qué se revisa |
|---|---|
| `cabecera:<nombre>` | Que salga en **toda** petición que el backend acota por ámbito, y qué pasa si falta |
| `parametro` | Lo mismo, y además que **ninguna pantalla lo tome de la URL sin pasar por el selector**: una URL compartida entre dos usuarios lleva el ámbito de quien la copió |
| `implicito-en-el-token` | Que al cambiar de ámbito se renueve el token, no solo la pantalla |
| `ninguno` | Que el backend lo derive del usuario **en cada endpoint**, no en algunos |

En los cuatro casos el valor es una **selección**, no una autoridad: el servidor tiene que cruzarlo
contra los ámbitos del usuario. Eso se comprueba desde el backend — punto 4b de
`aislamiento-de-datos` — y aquí solo se verifica que el frontend lo mande siempre y de un solo
lugar.

Y en los cuatro, la brecha más repetida:

> **Detalle contra listado.** El backend acota el listado por ámbito pero no el detalle por id. El
> frontend puede estar navegando a un detalle fuera de alcance desde un listado que sí filtró.
> Anota cada caso.

Además: ¿el selector de ámbito se oculta cuando solo hay uno? ¿Existe la opción "todos" y quién la
tiene?

---

## Cruce 5 · Bundle

Puntos binarios, se responden con un `grep` sobre `<raiz_frontend>`:

| # | Punto | Cómo se comprueba | BLOQUEA si |
|---|---|---|---|
| 1 | Cero inyección de HTML sin sanitizar | `grep -rn "dangerouslySetInnerHTML"` | Aparece con datos del backend o del usuario |
| 2 | Cero llamadas fuera del cliente central | `grep -rn "fetch(\|<frontend.libreria_http>"` excluyendo `frontend.cliente_http` | Aparece una |
| 3 | El access token no está en almacenamiento persistente | `grep -rn "localStorage"` | Aparece el token |
| 4 | Cero secretos o URLs privadas | `grep -rniE "api[_-]?key\|secret\|\.internal"` | Aparece uno |
| 5 | Ningún dato de prueba importado en código de producción | `grep -rniE "mock\|dummy\|seed\|demo"` | Un arreglo de prueba se importa desde una pantalla |
| 6 | Los enlaces y las imágenes con datos no ejecutan código | Guardar `javascript:alert(1)` en un campo de texto que la interfaz pinte como enlace o imagen, y abrir esa pantalla | El navegador lo ejecuta al hacer clic |

---

## Formato del reporte

`docs/00-brechas-frontend.md`, una sección por cruce. Cada hallazgo:

```
### F-<CRUCE>-<NN> · <título en una línea>
**Severidad:** P0 / P1 / P2
**Backend:**  <archivo:línea> — qué permite o exige
**Frontend:** <archivo:línea> — qué hace
**Qué ve el usuario:** <qué rol, en qué pantalla, con qué dato>
**Clase:** botón fantasma / función invisible / caché sucia / alcance de ámbito / bundle
```

## Reglas de esta auditoría

1. **No corriges código.** Documentas. Si arreglas mientras documentas, el reporte no describe ni el
   sistema viejo ni el nuevo.
2. **Sin evidencia de los dos lados no hay hallazgo.** Una sospecha va como `NO VERIFICABLE` con lo
   que haría falta para confirmarla.
3. **La severidad se justifica con el escenario, no con adjetivos.**
4. **Nada de diseño.** Espaciado, colores y consistencia visual no son esta auditoría.
5. Si el frontend y el backend difieren y **el frontend tiene razón** —el backend es más restrictivo
   de lo que el negocio quiere— dilo: es una brecha de producto, no un bug de interfaz.
