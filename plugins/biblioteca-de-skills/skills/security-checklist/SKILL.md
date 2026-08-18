---
name: security-checklist
description: >
  Checklist falsable de seguridad para APIs Django y REST Framework: autenticación, revocación de
  sesión, permisos por objeto, validación de entrada, secretos, CORS, límite de intentos, bitácora
  de auditoría, datos personales en logs y notificaciones. Cada punto se comprueba provocando un
  efecto, no leyendo una bandera de configuración. Úsala al diseñar el contrato, al implementar
  endpoints y obligatoriamente al revisar un módulo antes de fusionar. Requiere
  `protocolo-de-revision` y las claves `backend.*` y `cumplimiento.*` del perfil del repo.
---

# Checklist de seguridad

Carga antes `protocolo-de-revision`. Esta skill supone sus reglas puestas y no las repite.

## Claves del perfil que consume

`backend.autenticacion` · `backend.revocacion_de_sesion` · `backend.bitacora_auditoria` ·
`backend.borrado` · `frontend.almacen_token` · `cumplimiento.datos_sensibles` ·
`cumplimiento.monitoreo_errores` · `cumplimiento.consulta_legal` · `cumplimiento.registros_inmutables` ·
`cumplimiento.estados_con_razon` · `verificadores.entorno` · `verificadores.tests_backend` ·
`proyecto.raiz_backend` · `proyecto.raiz_frontend`

---

## La bandera que dice `True` y no hace nada

Esta lista existía antes en una versión que pedía *"revisar que `BLACKLIST_AFTER_ROTATION = True`"*.
Estaba en `True`. Y no servía de nada, porque la aplicación que implementa la revocación no estaba
instalada. Dos renglones antes, la misma lista pedía verificar que el logout invalidara el token —
lo contrario. Según cuál leyera el auditor, salía `PASA` o `BLOQUEA`.

Por eso **ningún punto de aquí se contesta abriendo `settings.py`**. Todos se contestan provocando
un efecto. Si al leer un punto puedes contestarlo sin ejecutar nada, el punto está mal escrito:
arréglalo y súbelo al plugin.

---

## 1 · Autenticación y sesión

| # | Punto | Cómo se comprueba | BLOQUEA si |
|---|---|---|---|
| 1 | Ningún endpoint responde sin credenciales | Llamar cada endpoint del módulo sin cabecera de autenticación | Alguno devuelve 200 |
| 2 | Un endpoint sin `permission_classes` no existe | Enumerar las vistas del módulo y llamar cada una con un usuario autenticado **sin** el rol requerido | Alguno devuelve 200 |
| 3 | **La sesión se puede revocar de verdad** | Cerrar sesión y reintentar con el token o la cookie anteriores | Responde 200 |
| 4 | El token no se puede leer desde un script de la página | `grep -rn "localStorage" <raiz_frontend>` y contrastar con `frontend.almacen_token` | El access token vive en `localStorage` |
| 5 | Las contraseñas se guardan cifradas de forma irreversible | Crear un usuario y leer el campo en la base | El valor es legible o descifrable |
| 6 | Las contraseñas débiles se rechazan | Intentar fijar `12345678` y el propio correo del usuario | Alguna se acepta |
| 7 | La fuerza bruta se detiene sola | Repetir el login fallido hasta pasar el umbral declarado | Nunca responde 429 ni bloquea |
| 8 | La cookie de sesión no es legible por JavaScript | Hacer login y leer la cabecera `Set-Cookie` de la respuesta | Falta `HttpOnly` o `SameSite` |

**Punto 3 es el que originó la Regla 1 del protocolo.** `backend.revocacion_de_sesion` en el perfil
debe haberse rellenado provocando este efecto. Si el perfil dice `ninguna`, este punto es
`BLOQUEA` conocido y ya debería estar en `docs/00-deuda.md` — no se vuelve a levantar en cada
revisión, pero tampoco se marca `PASA`.

**Punto 4 no tiene versión suave:** el access token vive en memoria y el refresh en
`sessionStorage` como mucho. El porqué, y la excepción de MVP que se borró en vez de degradarse,
están en `react-frontend`. Aquí solo el veredicto.

---

## 2 · Autorización

| # | Punto | Cómo se comprueba | BLOQUEA si |
|---|---|---|---|
| 9 | Ser dueño no se hereda de estar autenticado | Autenticar como usuario A y pedir por id un recurso de B | Devuelve 200 |
| 10 | El recurso ajeno no confirma su existencia | Mismo caso que el 9, mirando el código | Devuelve 403 en vez de 404 |
| 11 | El rol sale del servidor, no del cliente | Enviar `role`, `is_staff` o equivalente en el body de un PATCH del propio usuario | El rol cambia |
| 12 | Cada fila de la matriz de roles del contrato tiene su test | Contar filas de la matriz en `proyecto.contrato` contra tests existentes | Falta alguna |
| 13 | Los campos de estado no se cambian por PATCH genérico | Enviar `is_active`, `status` o equivalente en el PATCH de detalle | El valor cambia |

**Punto 13 — la puerta trasera del `is_active`.** Cambiar un estado tiene su propio endpoint y su
propio servicio (`*_desactivar`, `*_publicar`). Si viaja en el PATCH genérico, cualquier usuario con
permiso de edición se salta la regla de negocio que protege ese estado. Es de los tres bugs que
aparecieron repetidamente en revisiones reales.

**Punto 10** repite a propósito la decisión de `aislamiento-de-datos`: **403 = tu rol no puede hacer
esta acción · 404 = ese dato no existe para ti.** Aquí aplica entre usuarios del mismo ámbito; allá,
entre ámbitos.

---

## 3 · Entrada y salida

| # | Punto | Cómo se comprueba | BLOQUEA si |
|---|---|---|---|
| 14 | Ningún campo de texto es ilimitado | Enviar 1 MB de texto en cada campo libre del módulo | Se guarda, o devuelve 500 en vez de 400 |
| 15 | La entrada no llega a la consulta | Enviar `'; --` y `%` en cada campo de búsqueda | Error de SQL, o el `%` devuelve todo |
| 16 | Un archivo no es lo que dice su extensión | Subir un ejecutable renombrado a `.jpg`, uno que exceda el límite, y uno llamado `../../x.jpg` | Alguno se acepta o escapa del directorio |
| 17 | El error no es un mapa del sistema | Provocar un 500 con `DEBUG=False` y leer el cuerpo | Contiene traza, SQL o nombres de archivo |
| 18 | Los datos personales no se cachean en el navegador | Pedir un recurso con datos personales y leer las cabeceras de la respuesta | Falta `Cache-Control: no-store` |

**Punto 15:** el `%` está ahí por una razón distinta a la inyección. Un `icontains` sin escapar
convierte un `%` del usuario en comodín, y el listado devuelve todo lo que el filtro debía acotar.
No es inyección, es una fuga por búsqueda.

---

## 4 · Secretos

| # | Punto | Cómo se comprueba | BLOQUEA si |
|---|---|---|---|
| 19 | La aplicación no arranca sin sus secretos | Levantarla sin la variable de la llave secreta | Arranca con un valor por defecto |
| 20 | No hay secretos versionados, **tampoco en el historial** | `git ls-files \| grep -E '\.env'` y `git log -p -S "<fragmento de secreto>"` | Aparece uno, aunque ya esté borrado del árbol |
| 21 | El navegador de otro origen no obtiene respuesta | Petición con `Origin` ajeno a un endpoint autenticado | Devuelve `Access-Control-Allow-Origin: *` o refleja el origen |
| 22 | Ningún secreto viaja en un mensaje de commit | `git log --all --grep="<patrón de credencial>"` | Aparece uno |

**Punto 20 y 22:** un secreto que llegó a git ya es público aunque se borre. La revisión detecta;
el arreglo es **rotar la credencial**, no borrar el commit. Rotar va a `docs/05-despliegue.md`
porque no se puede comprobar desde el repo que se hizo.

---

## 5 · Datos personales

Esta sección entera se declara `N/A` **solo** si `cumplimiento.datos_sensibles: ninguno`. Con
cualquier otro valor, aplica completa.

| # | Punto | Cómo se comprueba | BLOQUEA si |
|---|---|---|---|
| 23 | El log no repite el dato sensible | Provocar un error sobre un registro sensible y leer el log completo | Aparece el dato o el objeto entero |
| 24 | El monitoreo de errores tampoco | Pasar por el filtro previo al envío un evento que contenga el dato sensible, y leer lo que sale | Sale con el dato, o no existe filtro |
| 25 | La notificación saliente lleva un identificador, no el contenido | Disparar cada notificación del módulo y leer el cuerpo emitido | Contiene el dato sensible |
| 26 | Queda registro de quién vio o cambió qué | Hacer la acción sensible y consultar `backend.bitacora_auditoria` | No hay registro, o no identifica al actor |
| 27 | El registro sensible no se borra físicamente | Borrar por la vía de la aplicación y buscar la fila en la base | La fila desapareció |
| 28 | Los registros que el perfil declara inmutables no se editan | Intentar un PATCH y un DELETE sobre cada modelo de `cumplimiento.registros_inmutables` ya emitido | Lo modifica o lo borra en sitio *(`N/A` si la clave dice `ninguno`)* |
| 29 | Las transiciones que el perfil marca exigen razón de lista cerrada | Para cada transición de `cumplimiento.estados_con_razon`: cambiarla sin razón, y con una razón inventada | Alguna se acepta *(`N/A` si la clave dice `ninguna`)* |

**Puntos 28 y 29 salen del perfil, no de aquí.** Nacieron como reglas duras de dos proyectos —*lo
clínico se corrige con addendum, nunca en sitio*; *todo cambio de estado de un lead exige una razón
de una lista cerrada*— y estaban escritas como si valieran en cualquier repo. Un sistema con datos
sensibles pero sin registros inmutables ni máquina de estados recibía dos BLOQUEA imposibles de
resolver y de declarar `N/A`. Ahora cada uno cita su clave.

**Punto 26 no se cumple con un log de texto.** Una bitácora que no se puede consultar por recurso y
por actor no responde la pregunta que la hace requisito: *quién vio este expediente*.

### Cumplimiento legal

Los requisitos legales aplicables **no se deducen de esta skill ni se le preguntan a un modelo**:
se consultan una vez con un abogado y el resultado se convierte en puntos fijos de esta lista.

Si `cumplimiento.consulta_legal: pendiente`, **esta lista no tiene puntos legales**. Eso significa
que la consulta sigue pendiente, no que no apliquen. Un modelo puede inventar un artículo con total
aplomo y nadie en el equipo tiene cómo detectarlo.

---

## Lo que salió de esta lista

Cuatro puntos de la versión anterior no sobrevivieron a la Regla 1. **No se suavizaron: se
mudaron.** Están en `docs/05-despliegue.md` con su verificación real.

| Punto anterior | Por qué no se puede comprobar desde el repo | Dónde quedó |
|---|---|---|
| `DEBUG = False` en producción, `ALLOWED_HOSTS` acotado | El valor efectivo es una variable de entorno del hosting; el repo solo tiene el default | Despliegue |
| HTTPS forzado, `SECURE_SSL_REDIRECT` y HSTS activos | Depende del proxy y del dominio, no del código | Despliegue |
| Cookie con `Secure` | Requiere HTTPS real para observarse | Despliegue *(la parte `HttpOnly`/`SameSite` sí se quedó: punto 8)* |
| Backups configurados y una restauración probada | Nada en el repo cambia si el backup no existe | Despliegue · D9 |
| Dependencias sin vulnerabilidades conocidas | El resultado depende del día, no del commit | Despliegue · D17 |
| Retención y acceso de los logs | Es configuración del proveedor | Despliegue · D20 |
| El correo saliente llega de verdad | El repo solo prueba que se encoló | Despliegue · D21 |
| Ninguna variable de producción con valor de desarrollo | El repo solo tiene el ejemplo, no el entorno | Despliegue · D3 |
| El monitoreo **recibe** eventos y no lleva datos personales | Configurado y sin eventos se ven igual desde el repo | Despliegue · D18, D19 |

El último era el mejor punto de toda la lista —*un backup nunca restaurado no es un backup*— y
**por eso mismo se mudó**: dejarlo aquí lo condenaba a contestarse con un `PASA` de memoria en cada
revisión. En la lista de despliegue lleva fecha de la última restauración probada, que es la única
forma de que sea cierto.

---

## Cuando algo falle en producción

El arreglo no termina en el commit. Termina cuando la causa se agrega como punto nuevo **de esta
lista en el plugin**, cumpliendo las dos reglas de admisión, para que la próxima revisión lo
atrape. Un incidente que no cambia el checklist está garantizado que se repite en el siguiente
proyecto.
