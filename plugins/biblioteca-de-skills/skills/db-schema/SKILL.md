---
name: db-schema
description: >
  Criterios de diseño de esquema y de consultas en Django y PostgreSQL: modelado, tipos correctos,
  relaciones y comportamiento al borrar, índices justificados por una consulta real, prevención de
  N+1, restricciones de unicidad en la base, migraciones reversibles y por pasos, borrado lógico y
  campos de auditoría. Úsala al diseñar el modelo de datos, al escribir una migración y al revisar
  consultas. Requiere `protocolo-de-revision` y las claves `proyecto.etapa`, `backend.borrado` y
  `verificadores.*` del perfil del repo.
---

# Diseño de esquema y consultas

Carga antes `protocolo-de-revision`. Esta skill supone sus reglas puestas y no las repite.

## Claves del perfil que consume

`proyecto.etapa` · `proyecto.raiz_backend` · `aislamiento.campo` · `backend.borrado` ·
`verificadores.entorno` · `verificadores.tests_backend` · `verificadores.migraciones`

---

## Modelado

- **Un modelo por concepto del negocio.** Si tienes que explicar un modelo con un "y también",
  probablemente son dos.
- Tipos correctos, no `CharField` para todo. **Dinero en `DecimalField`, nunca `FloatField`**:
  `0.1 + 0.2` no es `0.3`, y en un punto de venta eso es un descuadre de caja que aparece al tercer
  corte y nadie sabe explicar.
- `null=True` solo cuando *"no hay dato"* es distinto de *"está vacío"*. En texto, prefiere `""` a
  `NULL`: dos formas de decir "nada" es una fuente permanente de bugs, porque `filter(campo="")` no
  encuentra los `NULL` y `filter(campo__isnull=True)` no encuentra los vacíos.
- Todo modelo lleva `created_at` y `updated_at`. En datos sensibles, además quién lo creó.
- **Unicidad en la base de datos, no solo en el formulario.** La validación del serializer se salta
  con dos peticiones simultáneas; la restricción de la base, no.

## Relaciones y borrado

Elige `on_delete` con intención y escribe por qué:

| | Cuándo | Qué evita |
|---|---|---|
| `PROTECT` | El default sensato en datos de negocio | Borrar un producto que ya se vendió |
| `CASCADE` | Solo cuando el hijo no tiene sentido sin el padre | — (un renglón de una venta) |
| `SET_NULL` | La relación es informativa y puede perderse sin daño | — |

`CASCADE` por descuido es cómo se borra un histórico de ventas al eliminar un proveedor.

## Índices

**Un índice sin una consulta que lo necesite es peso muerto:** cada escritura lo actualiza.

Van donde hay una FK que se filtra seguido (el campo del ámbito, siempre), o un campo por el que se
busca u ordena en un listado real. En un índice compuesto **el orden de las columnas importa**:
primero el de igualdad, después el de rango.

Ejemplo concreto: el listado de ventas del punto de venta filtra por sucursal y por rango de fechas.
El índice es `(sucursal_id, vendida_en)`, en ese orden. Al revés sirve mucho menos.

## N+1

Listar 50 ventas dispara 51 consultas —una por la lista y una por cada venta al tocar sus
productos— y el usuario ve una pantalla lenta sin causa aparente.

`select_related()` para ForeignKey y OneToOne (hace JOIN). `prefetch_related()` para relaciones
inversas y ManyToMany (segunda consulta). **Un `for` que dentro accede a `objeto.relacion.campo` es
N+1 hasta que se demuestre lo contrario.**

## Migraciones

- **Reversibles.** Si una no lo es, se dice en el PR con letras grandes.
- Nunca renombrar y cambiar el tipo de una columna en la misma migración.
- Agregar una columna `NOT NULL` a una tabla con datos son **tres pasos**: agregar nullable,
  rellenar, volver obligatoria. Hacerlo en uno solo falla en producción y no en tu máquina, porque
  tu máquina no tiene datos.
- Migración de datos separada de la migración de estructura.

**Si `proyecto.etapa: produccion`, ninguna propuesta de cambio de esquema está completa sin decir
qué pasa con los datos que ya existen.** No es una formalidad: es la diferencia entre una migración
y una pérdida.

## Borrado lógico

En registros clínicos, ventas y facturas: marca de baja, nunca `DELETE`. Un borrado físico es
irreversible. Filtra los dados de baja en el manager por defecto para que no se cuelen en los
listados — y recuerda que entonces **el conteo del listado y el de la base dejan de coincidir**, lo
cual es correcto y hay que documentarlo.

---

## Revisión

| # | Punto | Cómo se comprueba | BLOQUEA si |
|---|---|---|---|
| 1 | El dinero no acumula error | Registrar tres importes de `0.10` y leer el total en la base | El total no es exactamente `0.30` |
| 2 | Borrar un padre con hijos hace lo que se decidió | Intentarlo en shell para cada FK nueva del módulo | El resultado no es el declarado (borra en cascada sin querer, o protege donde debía limpiar) |
| 3 | Cada índice nuevo lo usa una consulta real | `EXPLAIN` sobre la consulta del listado que lo justifica | El plan no lo usa |
| 4 | El número de consultas no crece con los datos | Contar consultas del listado con 10 registros y con 100 | El número sube |
| 5 | La unicidad aguanta dos peticiones a la vez | Dos inserciones concurrentes con el mismo valor | Las dos tienen éxito |
| 6 | Los campos de auditoría se llenan solos | Crear y luego modificar un registro; leer las dos marcas de tiempo | Alguna no cambia |
| 7 | El texto no tiene dos formas de estar vacío | `grep -rn "CharField\|TextField" <raiz_backend>` y revisar los `null=True` | Un campo de texto admite `NULL` y `""` |
| 8 | El registro que no puede perderse sigue ahí después de borrarlo | Borrar por la vía de la aplicación y buscar la fila en la base | La fila desapareció |
| 9 | Una columna obligatoria nueva no rompe con datos | Correr la migración contra una copia con datos reales | Falla |
| 10 | La migración se puede deshacer | Migrar hacia atrás a la revisión anterior | Falla y no está declarado como irreversible en el PR |

**Puntos 9 y 10 con `verificadores.migraciones: solo-emanuel`:** ningún agente corre `migrate`, así
que los dos salen `NO VERIFICABLE` y se mudan a `docs/05-despliegue.md`. Ahí es donde de verdad
importan — el día del despliegue, con el backup hecho.

---

## Lo que esta skill no puede verificar desde el repo

Va a `docs/05-despliegue.md`:

- Que exista un backup **inmediatamente antes** de correr la migración, y que se sepa cómo volver.
- Que el plan de consultas en producción se parezca al de desarrollo. Una tabla con 200 filas usa
  otro plan que la misma tabla con 200 000: el índice que aquí no se nota, allá es la diferencia
  entre 40 ms y 12 s.
