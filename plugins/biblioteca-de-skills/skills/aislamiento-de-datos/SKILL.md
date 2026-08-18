---
name: aislamiento-de-datos
description: >
  Cómo se garantiza que los datos de un ámbito no lleguen a otro, sea ese ámbito un tenant (varios
  clientes en la misma base) o una sede (varias sucursales de un mismo negocio). Úsala al diseñar el
  modelo de datos, al implementar cualquier endpoint que devuelva datos, y obligatoriamente al
  revisar. Trae el test de fuga con 404, la regla de que el ámbito nunca viene del cliente, el
  manager filtrado por defecto y la variante para repos que filtran vista por vista. Requiere
  `protocolo-de-revision` y las claves `aislamiento.*` del perfil del repo.
---

# Aislamiento de datos

Carga antes `protocolo-de-revision`. Esta skill supone sus reglas puestas y no las repite.

## Claves del perfil que consume

`aislamiento.ambito` · `aislamiento.nombre_de_negocio` · `aislamiento.campo` ·
`aislamiento.mecanismo` · `aislamiento.modelo_base` · `aislamiento.escape` · `aislamiento.origen` ·
`aislamiento.test_de_fuga` · `aislamiento.ancla` · `backend.capa_de_servicios` ·
`verificadores.tests_backend` · `verificadores.entorno`

**Si `aislamiento.ambito` no está en el perfil, esta skill entera se marca `NO VERIFICABLE` y se
detiene.** No se infiere del código: un repo puede tener un campo `sede` en cada modelo y no estar
aislando nada por él.

---

## Principio

> **El aislamiento no puede depender de que el programador se acuerde de filtrar.**
>
> Cualquier diseño cuya seguridad se apoye en la disciplina humana falla el día que alguien tiene
> prisa. El mecanismo debe hacer que *olvidar* el filtro sea imposible, o que rompa ruidosamente.

Este principio no depende del ámbito. Vale igual para *"el expediente de la clínica A no llega a la
clínica B"* que para *"la venta de la sucursal Norte no aparece en el corte de Centro"*. La única
diferencia entre los dos casos es la palabra, y esa palabra está en `aislamiento.ambito`.

**Un repo multi-sucursal no está exento de esta skill.** `ambito: ninguno` es la única forma
legítima de declararla `N/A`, y significa que el sistema tiene un solo conjunto de datos sin
compartimentos — no que los compartimentos se llamen distinto.

---

## Los dos mecanismos, y qué cambia entre ellos

| | `manager-por-defecto` / `rls` | `manual-por-vista` |
|---|---|---|
| Dónde está el filtro | Uno, en el modelo o en la base | Repetido en cada vista |
| Qué falla cuando alguien olvida | Nada: no se puede olvidar | Esa vista, en silencio |
| Cómo se audita | Se revisa el mecanismo, una vez | Se revisa **cada endpoint**, con `aislamiento.ancla` |
| Coste de la revisión | Constante | Crece con el número de endpoints |

**Lo que no cambia entre los dos es el test de fuga.** El mecanismo es una decisión de arquitectura;
el efecto observable es el mismo, y es el efecto lo que se revisa.

### El mecanismo preferido

```python
class AmbitoManager(models.Manager):
    def get_queryset(self):
        ambito = get_current_ambito()
        if ambito is None:
            raise AmbitoNoEstablecido("Acceso a datos sin contexto de ámbito")
        return super().get_queryset().filter(**{f"{CAMPO}_id": ambito.id})
```

Tres piezas, y las tres son necesarias:

1. **Contexto por petición.** El ámbito se resuelve donde diga `aislamiento.origen` y se deja en un
   contextvar. Si no se puede resolver, **la petición se rechaza** — nunca "sigue sin ámbito".
2. **El manager por defecto (`objects`) ya trae el filtro puesto.** El olvido deja de ser posible en
   el camino normal.
3. **Un manager de escape, vigilado.** El del perfil (`aislamiento.escape`), sin filtro, **solo**
   para migraciones, comandos de administración y tareas programadas. Cada uso lleva comentario.

### ⚠ Cambiar de `manual-por-vista` a `manager-por-defecto` no es un gate

En un repo con `proyecto.etapa: produccion`, cambiar el manager por defecto de modelos con datos
vivos **altera en silencio el resultado de comandos, seeds y reportes**. Un corte de caja que ayer
sumaba todas las sucursales hoy suma una, sin error y sin aviso.

Va como **objetivo con plan de migración**, no como BLOQUEA. La revisión anota la brecha; el cambio
se planea aparte, modelo por modelo, con los comandos y reportes revisados uno por uno.

---

## Reglas duras

**El cliente elige; el servidor autoriza.** Son dos cosas distintas y confundirlas es la falla:

- **Autoridad — qué ámbitos puede ver este usuario.** Sale de `aislamiento.origen`, siempre del
  servidor: un campo del usuario, un claim del token, una tabla de membresías, el subdominio. Nunca
  de algo que el cliente escriba.
- **Selección — cuál de esos ámbitos está usando ahora.** Esto el cliente **sí** puede decirlo, por
  donde diga `frontend.transporte_del_ambito`. Y el servidor **lo cruza contra los autorizados
  antes de usarlo.**

Un servidor que toma el ámbito del cliente sin cruzarlo contra los suyos no tiene aislamiento: tiene
un campo de formulario con el id de otra clínica. Un servidor que no deja al cliente elegir obliga a
un usuario con dos sedes a cerrar sesión para cambiar de sede.

**`on_delete=PROTECT` en la FK al ámbito.** Borrar un ámbito no debe arrastrar sus datos en cascada
por accidente.

**Toda lectura de un objeto por id pasa por un selector.** Un `Model.objects.get(id=...)` directo
en la vista es la forma canónica de saltarse el filtro. Y es la misma decisión que el 404 de abajo,
mirada desde el otro lado: si la consulta filtra por ámbito *antes* de buscar por id, el 404 sale
solo. Por eso un repo con `backend.capa_de_servicios: ninguna` filtra mal por diseño, no por
descuido.

---

## 404, no 403

> **403 = tu rol no puede hacer esta acción · 404 = ese dato no existe para ti.**

Un 403 sobre un recurso ajeno **confirma que el recurso existe**. Parece una fuga pequeña y no lo
es: permite contar por enumeración de ids cuántas ventas, cuántas recetas o cuántos pacientes hay
en otro ámbito, sin ver un solo campo. Sobre datos de salud, confirmar la existencia ya es
información.

Se ofreció la salida fácil —403 para todo, un solo camino de error en el frontend— y **se rechazó a
propósito**. No cuesta código: si la consulta filtra por ámbito antes de buscar por id, el registro
no está en el conjunto y no hay nada que fingir.

---

## El test de fuga

Sin este test, el módulo no pasa revisión. No hay versión reducida.

```python
def test_no_puede_leer_datos_de_otro_ambito(api_client, ambito_a, ambito_b):
    registro_b = RegistroFactory(ambito=ambito_b)
    api_client.force_authenticate(UserFactory(ambito=ambito_a))

    respuesta = api_client.get(f"/api/registros/{registro_b.id}/")

    assert respuesta.status_code == 404          # 404, no 403
```

Una variante por cada endpoint que devuelva datos del ámbito. Es repetitivo y por eso vale la pena
parametrizarlo — pero no se omite.

**Y la segunda mitad, que casi siempre falta:** el listado.

```python
def test_el_listado_no_contiene_datos_de_otro_ambito(api_client, ambito_a, ambito_b):
    RegistroFactory(ambito=ambito_b)
    api_client.force_authenticate(UserFactory(ambito=ambito_a))

    ids = [r["id"] for r in api_client.get("/api/registros/").json()["..."]]

    assert str(registro_b.id) not in ids
```

El detalle y el listado fallan por caminos distintos. **La brecha más repetida es un listado que sí
filtra y un detalle por id que no**, porque el detalle se escribió después y nadie volvió al test.

---

## Revisión

Cada punto se responde según `protocolo-de-revision`. `<escape>`, `<campo>` y `<ancla>` salen del
perfil. Los comandos van precedidos de `verificadores.entorno`.

| # | Punto | Cómo se comprueba | BLOQUEA si |
|---|---|---|---|
| 1 | Existe el test de fuga por id y pasa | Correr `verificadores.tests_backend`; localizar el test en `aislamiento.test_de_fuga` | No existe, no pasa, o afirma 403 en vez de 404 |
| 2 | Existe el test de fuga del listado y pasa | Ídem, sobre el endpoint de lista | No existe o el listado devuelve un id ajeno |
| 3 | Hay test de fuga para **cada** endpoint nuevo del módulo | Contar endpoints del módulo contra variantes del test | Falta uno |
| 4 | El ámbito no se acepta desde el body | Enviar `<campo>` con el id de otro ámbito en el body de un POST y un PATCH | El registro queda en el ámbito enviado |
| 4b | La selección de ámbito se cruza contra los autorizados | Enviar por `frontend.transporte_del_ambito` el id de un ámbito **al que el usuario no pertenece**, y pedir un listado | Devuelve datos de ese ámbito |
| 5 | Sin contexto de ámbito, el acceso revienta | En shell, consultar el modelo fuera de una petición | Devuelve datos en vez de lanzar excepción *(N/A si `mecanismo: manual-por-vista`)* |
| 6 | El escape está acotado | `grep -rn "<escape>" <raiz_backend>` y clasificar cada uso | Aparece fuera de migraciones, comandos o tareas, o sin comentario |
| 7 | Cada endpoint del módulo filtra por ámbito | `<ancla>` y revisar uno por uno *(solo con `mecanismo: manual-por-vista`)* | Un endpoint que devuelve datos no filtra |
| 8 | Las consultas crudas filtran explícitamente | `grep -rn "\.raw(\|\.extra(\|cursor.execute" <raiz_backend>` y leer cada una | Una no lleva el filtro del ámbito |
| 9 | Detalle y listado tienen el mismo alcance | Pedir por id un registro que el listado del usuario no muestra | El detalle lo devuelve |
| 10 | La FK al ámbito es `PROTECT` | En shell, borrar un ámbito con registros hijos | Borra en cascada en vez de lanzar `ProtectedError` |
| 11 | Los archivos subidos van a una ruta con el ámbito | Subir un archivo y leer la ruta resultante | La ruta no distingue ámbitos |
| 12 | Si el mecanismo incluye `rls`: cada modelo nuevo trae su migración | Correr el test guardián de cobertura de RLS | Falla y se silenció en vez de agregar la migración |

**Punto 12, sobre el patrón general:** cuando dos fuentes definen el mismo requisito de aislamiento
—un modelo base, un manager y una migración de RLS con test guardián— manda la que trae el
verificador. Las otras apuntan a ella. Una regla sin verificador es una intención.

---

## Lo que esta skill no puede verificar desde el repo

Va a `docs/05-despliegue.md`:

- Que el rol de base de datos con el que corre la aplicación en producción **no sea superusuario**.
  Con RLS activo y un rol superusuario, las políticas no se aplican y todos los tests siguen verdes.
- Que la restauración de un backup conserve las políticas de aislamiento, no solo las tablas.
