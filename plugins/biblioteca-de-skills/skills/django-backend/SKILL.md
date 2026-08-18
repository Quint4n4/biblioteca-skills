---
name: django-backend
description: >
  Convenciones de código para Django y Django REST Framework: apps por dominio de negocio, capa de
  servicios y selectores, serializers por caso de uso, tipado, forma de los errores, paginación,
  atomicidad y concurrencia en operaciones que escriben, y tests con pytest. Úsala al implementar o
  al revisar cualquier código de backend en este stack. No cubre seguridad ni aislamiento de datos:
  eso vive en `security-checklist` y `aislamiento-de-datos`. Requiere `protocolo-de-revision` y las
  claves `backend.*` del perfil del repo.
---

# Convenciones de backend

Carga antes `protocolo-de-revision`. Esta skill supone sus reglas puestas y no las repite.

**Esta skill no habla de seguridad ni de aislamiento.** Tuvo secciones de las dos y las duplicaba
peor que las skills dedicadas, con el resultado de que el mismo requisito existía en tres versiones
distintas. Entre dos definiciones de la misma regla gana la que trae el verificador; las de aquí no
lo traían y se borraron.

## Claves del perfil que consume

`backend.framework` · `backend.forma_de_vistas` · `backend.capa_de_servicios` ·
`backend.envoltura_respuesta` · `backend.paginacion` · `backend.tareas_asincronas` ·
`aislamiento.ambito` · `aislamiento.campo` · `verificadores.entorno` · `verificadores.tests_backend` ·
`verificadores.tipos` · `verificadores.lint` · `proyecto.raiz_backend`

---

## Estructura

Apps por **dominio de negocio** (`inventory`, `billing`, `appointments`), nunca por capa técnica
(`api`, `models`, `utils`). Cuando un día quites la facturación quieres borrar una carpeta, no
perseguir diez archivos.

```
<dominio>/
├── models.py       # estructura y propiedades derivadas. Sin lógica de negocio.
├── selectors.py    # lecturas: funciones que devuelven querysets o datos
├── services.py     # escrituras y reglas de negocio. Las transacciones viven aquí.
├── serializers.py  # uno por caso de uso
├── views.py        # valida entrada, llama a un service o selector, responde
├── permissions.py
└── tests/
```

**La vista no decide.** Si tiene un `if` sobre una regla del negocio, ese `if` va en `services.py`.
El nombre formal del patrón es *service layer*; el beneficio concreto es que la misma regla sirve
para un endpoint, un comando de consola y una tarea programada sin duplicarse — y que se puede
probar sin levantar HTTP.

`backend.forma_de_vistas` decide la forma, no el reparto. Un repo con `apiview-y-path` no introduce
un router suelto porque una skill muestre un `ViewSet`; un repo con `viewsets-y-routers` no baja a
`path()` por lo mismo. **Sigue lo que diga el perfil.**

---

## Servicios y selectores

```python
@transaction.atomic
def stock_transfer(*, product: Product, origin: Branch, target: Branch,
                   quantity: int, user: User) -> StockTransfer:
    """Traspasa existencias entre sucursales. Lanza ValidationError si no alcanza."""
```

- **Argumentos con nombre obligatorio (`*`).** En la llamada se lee qué es cada cosa.
- **Tipado en toda firma**, argumentos y retorno. Con verificador de tipos o sin él, el tipo es
  documentación que no envejece.
- Se nombran como acción + entidad: `appointment_create`, `patient_update`, `invoice_get`.
- **Toda lectura por id vive en un selector.** Un `Model.objects.get(id=...)` en la vista es la
  forma canónica de saltarse el filtro de ámbito — ver `aislamiento-de-datos`.
- Validación de **forma** en el serializer; validación de **reglas de negocio** en el service.

### Campos que un update genérico nunca toca

```python
_INMUTABLES: frozenset[str] = frozenset(
    {"id", "created_at", "updated_at"}
    # + el campo de baja del repo, el flag de estado, `aislamiento.campo`
    # y las FK de identidad. Los nombres exactos salen del perfil, no de aquí.
)
```

El servicio de update rechaza explícitamente cualquiera de estos. Sin esa lista, un PATCH reactiva un registro dado de baja
saltándose la regla de negocio que lo dio de baja.

---

## Atomicidad y concurrencia

Esta sección no venía de una skill: **venía del código.** El servicio de descuento de stock FEFO de un POS de farmacia
ordena los lotes por caducidad, excluye los caducados y toma `select_for_update()` antes de
descontar, todo dentro de una operación que toca varias tablas. Ninguna de las skills anteriores
mencionaba atomicidad ni concurrencia, pese a definir una capa de servicios cuya razón de ser es
escribir.

**`@transaction.atomic` en toda operación que escriba en más de una tabla.** Un traspaso que
descuenta de una sucursal y falla al sumar en la otra deja mercancía inexistente, y nadie se entera
hasta el inventario físico.

**`select_for_update()` cuando dos peticiones pueden competir por el mismo registro.** El caso real:
queda una caja del último lote y dos cajeras cobran a la vez. Sin bloqueo, las dos leen "1
disponible", las dos descuentan, y el stock queda en −1 con dos tickets emitidos.

```python
@transaction.atomic
def stock_take(*, product: Product, quantity: int) -> None:
    lotes = (
        Lote.objects.select_for_update()          # bloquea las filas hasta el commit
        .filter(product=product, expires_at__gt=today())   # los caducados no se venden
        .order_by("expires_at")                            # primero el que vence antes
    )
```

Tres detalles que se olvidan por separado: el `select_for_update` va **dentro** del `atomic` o no
hace nada; el orden decide cuál lote se consume; y el filtro de caducidad tiene que estar en la
consulta, no en un `if` posterior — si está después, el bloqueo ya se tomó sobre filas que no se
iban a usar.

`backend.tareas_asincronas: ninguna` significa que la operación larga corre en el hilo de la
petición. Con pocos usuarios concurrentes está bien y **no se introduce una cola sin necesidad
demostrada**; introducirla es una decisión de arquitectura, no una mejora automática.

---

## Serializers

Uno por caso de uso: `ProductListSerializer`, `ProductDetailSerializer`, `ProductCreateSerializer`.
Reutilizar el de escritura para lectura termina exponiendo campos internos el día que alguien
agrega uno.

Entrada y salida **separadas**, aunque hoy tengan los mismos campos. El día que divergen, divergen
sin avisar.

Los serializers no llevan lógica de negocio en `create()` ni en `update()`: eso es el service.

---

## Errores y paginación

- La forma de la respuesta de error la fija `backend.envoltura_respuesta` y se respeta en toda la
  API. Un endpoint que responde distinto obliga al frontend a un caso especial permanente.
- `ValidationError` de Django o DRF, nunca `Exception` genérica.
- Nunca devolver el mensaje interno de una excepción al cliente.
- **Paginación en todo listado**, con la forma que diga `backend.paginacion`. Sin ella, el listado
  que hoy tiene 30 filas mañana tiene 30 000 y tumba la pantalla.

---

## Tests

- Factories, no fixtures JSON.
- Un test por camino feliz, al menos uno por caso borde y uno por permiso denegado.
- Nombres descriptivos en español: `def test_no_permite_traspaso_con_stock_insuficiente():`
- Contar consultas en los listados, para que un N+1 futuro rompa el test.
- El test de fuga de ámbito es obligatorio si `aislamiento.ambito` no es `ninguno`.

---

## Revisión

| # | Punto | Cómo se comprueba | BLOQUEA si |
|---|---|---|---|
| 1 | Las apps agrupan dominios, no capas | `grep -rn "def \|class " <raiz_backend>/{api,core,utils,common}/` y clasificar lo que salga | Aparece una regla de negocio dentro de una app de capa técnica |
| 2 | La vista no decide | `grep -rn "if " <raiz_backend>/*/views.py` y, por cada uno, llamar el mismo caso de uso desde el shell **a través del service** | El service acepta lo que la vista rechaza: la regla solo existe en la vista |
| 3 | Una operación multi-tabla no deja estado a medias | Forzar una excepción entre las dos escrituras y consultar las dos tablas | Una quedó escrita y la otra no |
| 4 | Dos escrituras simultáneas no duplican el último recurso | Dos transacciones concurrentes tomando la última unidad | Las dos tienen éxito |
| 5 | Todo listado viene acotado | Crear 1 000 registros y pedir el listado sin parámetros | Devuelve los 1 000 |
| 6 | Los errores tienen una sola forma | Provocar 400, 403, 404 y 500 y comparar los cuerpos | Alguno se sale de `backend.envoltura_respuesta` |
| 7 | Ningún serializer de escritura se usa para leer | `grep -rn "Serializer(\|serializer_class" <raiz_backend>` y, por cada clase que aparezca en un POST/PATCH y en un GET, comparar los campos que devuelven | La misma clase sirve los dos sentidos |
| 8 | Un update genérico no cambia campos de identidad ni de estado | Enviar `id`, el campo del ámbito y el flag de estado en un PATCH | Alguno cambia |
| 9 | Las lecturas por id pasan por selector | `grep -rn "objects.get(" <raiz_backend>` en `views.py` | Aparece en una vista |
| 10 | Los tipos declarados son ciertos y no queda código muerto | Correr `verificadores.tipos` y `verificadores.lint` | Alguno falla |
| 11 | Hay test de camino feliz, borde y permiso denegado por caso de uso nuevo | Correr `verificadores.tests_backend` y contrastar con los servicios del módulo | Falta alguno de los tres |

**Punto 4 en detalle**, porque es el que nadie escribe. Dos conexiones, dos transacciones abiertas a
la vez, las dos intentando consumir la última unidad. Con `select_for_update()` la segunda espera y
después falla por falta de existencia. Sin él, las dos leen el mismo número y las dos cobran. Si
`verificadores.tests_backend: ninguno`, este punto sale `NO VERIFICABLE` a la lista de despliegue —
no `PASA`.

---

## Lo que esta skill no puede verificar desde el repo

- Que el número de usuarios concurrentes reales justifique (o no justifique) mover una operación
  larga a una cola. Se decide con una medición en producción, no con una opinión.
