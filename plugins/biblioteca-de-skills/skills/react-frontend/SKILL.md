---
name: react-frontend
description: >
  Cómo se conecta un frontend React + TypeScript a una API Django REST: cliente HTTP central,
  manejo de tokens, refresh en 401, distinción entre 403 y 404, tipos derivados de la API, mapeo de
  errores de validación a los campos del formulario, paginación y seguridad del bundle. Úsala al
  escribir o revisar cualquier código de frontend que hable con el backend. Requiere
  `protocolo-de-revision` y las claves `frontend.*` del perfil del repo. Para auditar un frontend ya
  escrito contra el contrato, usa `auditoria-frontend`.
---

# Frontend contra la API

Carga antes `protocolo-de-revision`. Esta skill supone sus reglas puestas y no las repite.

## Claves del perfil que consume

`frontend.apps` · `frontend.estado_servidor` · `frontend.estilos` · `frontend.cliente_http` ·
`frontend.almacen_token` · `frontend.libreria_http` · `frontend.transporte_del_ambito` · `backend.autenticacion` ·
`backend.envoltura_respuesta` · `backend.paginacion` · `verificadores.tipos` ·
`verificadores.tests_frontend` · `proyecto.raiz_frontend`

**Esta skill no elige la librería.** Estado de servidor, estilos y forma del cliente salen del
perfil. Si el perfil dice `estado_servidor: ninguno — axios + useState`, **no metas una librería de
caché en un archivo suelto**: o se migra completo con decisión explícita, o se sigue con lo que hay.
Mezclar dos formas de pedir datos en el mismo repo cuesta más que cualquiera de las dos.

---

## Las cuatro reglas de oro

**1 · El backend es la autoridad de permisos.** Lo que el frontend decide es qué se *muestra*. Nunca
es seguridad. Ocultar un botón no protege el endpoint: protege al usuario de un error, y eso es
todo. El backend ya responde 403.

**2 · Cero secretos en el bundle.** Todo lo que se compila es público, incluidas las variables de
entorno del compilador. Nada de llaves de terceros ni tokens de servicio: solo la URL de la API y
configuración pública.

**3 · Los tipos de la API son ciertos o no son tipos.** Estricto, sin `any`. Preferido: derivarlos
del esquema del backend, para que un cambio del backend rompa la compilación en vez de romper una
pantalla. Alternativa: escritos a mano y mantenidos en sincronía a propósito.

**4 · Toda llamada pasa por el cliente HTTP central.** El módulo que dice `frontend.cliente_http` es
el único que hace peticiones de red (`frontend.libreria_http`), la URL base, los tokens, el refresh y la forma del error. Una
llamada suelta en un componente es una excepción a las cuatro reglas a la vez.

---

## Capas

```
Componente (UI)
   └─ estado de servidor            ← frontend.estado_servidor
        └─ función de API tipada     ← una por endpoint, devuelve datos, no respuestas crudas
             └─ cliente HTTP central ← frontend.cliente_http
                  └─ la API
```

Con `estado_servidor: ninguno` la segunda capa desaparece y su trabajo —cache, invalidación,
recarga— pasa a los componentes. **La tercera capa no desaparece nunca.** Es la que hace que un
cambio de endpoint se arregle en un archivo.

---

## Tokens

> **El access token vive en memoria. El refresh, en `sessionStorage` como mucho. Nunca en
> `localStorage`.**

No hay versión para MVP. Hubo una versión de esta biblioteca que decía que `localStorage` *"es
aceptable si y solo si el frontend no tiene vectores XSS"* y traía ejemplo de código. Un ejemplo de
código es lo que un agente copia, y la condición —"si y solo si no hay XSS"— no se puede comprobar
el día que se escribe, solo el día que falla.

- Un solo módulo toca el almacenamiento del token. Ninguno más.
- El token **nunca** en un `console.log`, en la URL, ni en un mensaje de error.
- Al cerrar sesión: limpiar el token **y** el caché de datos. Ver `auditoria-frontend`, cruce 3.

---

## 401, 403 y 404

| Código | Qué significa | Qué hace el frontend |
|---|---|---|
| 401 | La sesión ya no vale | Refrescar **una vez**; si falla, limpiar y mandar a login |
| 403 | **Tu rol no puede hacer esta acción** | Mensaje claro; no romper la pantalla |
| 404 | **Ese dato no existe para ti** | Tratarlo como "no encontrado", nunca como error del sistema |

Las tres reglas que se rompen:

- **El refresh se reintenta una sola vez**, y nunca sobre el propio endpoint de refresh. Sin ese
  tope es un bucle infinito que el usuario ve como una pantalla congelada.
- **El 403 no se traga.** Se propaga y se muestra. Idealmente la UI ya escondió esa acción, pero el
  403 es la red de seguridad y silenciarlo convierte un permiso mal puesto en un botón que no hace
  nada.
- **El 404 de un recurso ajeno no es un fallo.** El backend responde 404 en vez de 403 para no
  confirmar que el registro existe. Si el frontend lo pinta como "error de servidor", el usuario
  llama a soporte por una respuesta correcta.

---

## Errores de validación y paginación

- El error de validación se **mapea a su campo** en el formulario. Un 400 que aparece como mensaje
  genérico obliga al usuario a adivinar cuál de ocho campos está mal.
- La forma exacta del error y del listado sale de `backend.envoltura_respuesta` y
  `backend.paginacion`. Si la envoltura no es la plana del framework, **todas** las funciones de API
  la desenvuelven en el mismo lugar, no cada componente por su cuenta.
- **Nunca asumir que una lista viene completa.** El backend pagina. Un componente que hace
  `.length` sobre la primera página y lo llama "total" es un número mal en pantalla.
- Toda lista tiene estado de carga y estado vacío. "Sin resultados" y "todavía cargando" no se ven
  igual.

---

## Revisión

| # | Punto | Cómo se comprueba | BLOQUEA si |
|---|---|---|---|
| 1 | No hay llamadas fuera del cliente central | `grep -rn "fetch(\|<frontend.libreria_http>" <raiz_frontend>` excluyendo `frontend.cliente_http` | Aparece una |
| 2 | El token no está en almacenamiento persistente | `grep -rn "localStorage" <raiz_frontend>` | El access token aparece ahí |
| 3 | El token no se filtra por consola ni por URL | `grep -rn "console.log" <raiz_frontend>` y revisar los que tocan sesión | Un token acaba impreso o en un query param |
| 4 | El refresh no entra en bucle | Invalidar la sesión en el servidor y usar la aplicación | Se repite la petición de refresh sin fin |
| 5 | El 403 se muestra, no se traga | Entrar con un rol sin permiso y forzar la acción | La pantalla se rompe o no dice nada |
| 6 | El 404 no se pinta como error del sistema | Pedir por URL el id de un recurso de otro usuario | Muestra "error de servidor" |
| 7 | El rol sale del servidor | `grep -rn "rol\|role" <raiz_frontend>` buscando valores fijos | Hay un rol escrito a mano en el código |
| 8 | Los errores de validación llegan a su campo | Enviar un formulario con dos campos inválidos | El mensaje sale genérico o en el campo equivocado |
| 9 | Los listados no asumen array completo | Cargar una lista con más registros que el tamaño de página | El total mostrado es el de la página |
| 10 | Los tipos compilan | Correr `verificadores.tipos` | Falla |
| 11 | No hay secretos en el código del cliente | `grep -rniE "api[_-]?key\|secret\|token=" <raiz_frontend>` | Aparece uno |
| 12 | No se mezclan dos formas de pedir datos ni de estilar | `grep -rhn "^import" <archivos nuevos del módulo>` y contrastar la lista contra `frontend.estado_servidor` y `frontend.estilos` | Aparece una librería que el perfil no declara |

Los puntos 4, 5, 6, 8 y 9 se comprueban usando la aplicación. Si
`verificadores.tests_frontend: ninguno`, **siguen siendo comprobables a mano** y se comprueban a
mano: son cinco minutos y son los cinco que más soporte generan. No salen a la lista de despliegue.
