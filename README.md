Aquí va la HU-03 en el mismo formato que la HU-02. Separé en dos pasos lo que hace el empleado: primero **buscar** la empresa en COBIS y después **vincularla**.

---

## 🟢 HU-03 · Vincular empresa al grupo económico
**Microservicio:** `ms-grupos-economicos`
**Endpoints:** `GET /v1/empresas-core?identificacion={ruc}` (buscar) · `POST /v1/grupos-economicos/{id}/empresas` (vincular)

**Como** empleado de Servicio de Canales, **quiero** buscar una empresa en COBIS y asociarla a un grupo económico, **para** que después se vinculen sus productos y se invite a sus usuarios.

### Componentes
| Componente | Qué hace en esta historia |
|---|---|
| **BackOffice web** (pantalla del banco) | Busca por RUC o cédula, muestra lo que devuelve COBIS y el empleado confirma |
| **API Gateway** | Valida el token del empleado y enruta al microservicio |
| **ms-grupos-economicos** | Consulta COBIS, valida las reglas, guarda la empresa y registra el evento |
| **Integración con COBIS** (servicio del core, o un mock al inicio) | Devuelve el id, el nombre y el tipo de persona de la empresa. **Solo lectura** |
| **Base BELC (Aurora MySQL)** | Tablas `Grupo_economico`, `Empresas_asociadas`, `BITACORA_EVENTO` y `TIPO_EVENTO` |
| **Librería de bitácora** (HU-13) | Registra el evento en la misma transacción |
| ForgeRock | **No interviene** |

### Modelo de datos
**`Grupo_economico`** (solo lectura): se verifica que el `id_grupo_economico` de la URL exista.

**`Empresas_asociadas`** (inserta una fila)
| Campo | De dónde viene |
|---|---|
| `id_empresa_asociada` | Lo genera la base (PK) |
| `id_grupo_economico` | De la URL (FK → `Grupo_economico`) |
| `Nombre` | De COBIS (no lo escribe el empleado) |
| `Tipo_persona` | De COBIS: `PJ` (jurídica) o `PN` (natural) |
| `core_empresasoc_id` | El id de la empresa en COBIS |

**`BITACORA_EVENTO`** (inserta una fila)
| Campo | Valor |
|---|---|
| `id_tipo_evento` | → `TIPO_EVENTO` con código `EMPRESA_VINCULADA` y categoría `BACKOFFICE` |
| `Resultado` / `Canal` | `Exito` o `Fallo` / `BO` |
| `id_grupo_economico` | El grupo |
| `Usuario_backoffice` | El empleado |
| `Entidad`, `id_registro` | `Empresas_asociadas` y su id |
| `Valor_nuevo` | El JSON con nombre, tipo de persona y `core_empresasoc_id` |
| `Codigo_error` | Se llena si COBIS falló |
| `id_correlacion`, `Fecha_evento_utc` | El `correlationId` y la fecha en UTC |

### Contrato
**1. Buscar en COBIS:** `GET /v1/empresas-core?identificacion=155612345-2-2019`
```json
{ "coreEmpresaId": "COBIS-EMP-66001", "nombre": "Constructora Pacífico, S.A.", "tipoPersona": "PJ", "yaVinculada": false }
```

**2. Vincular:** `POST /v1/grupos-economicos/20/empresas`
```json
{ "coreEmpresaId": "COBIS-EMP-66001" }
```
**Respuesta 201**
```json
{ "idEmpresaAsociada": 201, "idGrupoEconomico": 20, "nombre": "Constructora Pacífico, S.A.", "tipoPersona": "PJ", "coreEmpresaId": "COBIS-EMP-66001" }
```
**Errores:** `400` datos inválidos · `403` sin permiso · `404` el grupo no existe, o la empresa no existe en COBIS · `409` la empresa ya está vinculada a un grupo · `502` COBIS respondió con error · `504` COBIS no respondió a tiempo.

### Flujo
1. El empleado busca la empresa por su identificación, y el microservicio consulta COBIS.
2. La pantalla muestra el nombre y el tipo de persona. Si la empresa ya está vinculada, lo indica.
3. El empleado confirma, y el microservicio **vuelve a consultar COBIS** con el `coreEmpresaId`. No se confía en lo que manda la pantalla.
4. Valida que el grupo exista y que la empresa no esté en otro grupo.
5. En **una sola transacción**: inserta en `Empresas_asociadas` y en `BITACORA_EVENTO`.
6. Responde 201.

### Criterios de aceptación
1. Guarda la empresa con su `core_empresasoc_id`, y su nombre y tipo de persona tal como vienen de COBIS.
2. Una empresa no puede estar en dos grupos: si ya está vinculada, responde 409.
3. Si COBIS no responde, devuelve 504 y no guarda nada. El intento queda en la bitácora con resultado `Fallo`.
4. Si el grupo no existe, responde 404.
5. BELC solo **lee** de COBIS; no crea ni modifica nada en el core.
6. Mientras el servicio real no esté disponible, funciona con un mock de COBIS que devuelve el mismo formato.

### Pendientes en tu modelo (revisados en las fotos)
- **Falta un UNIQUE en `Empresas_asociadas.core_empresasoc_id`.** Sin él, el criterio 2 depende solo del código.
- **`Empresas_asociadas` no tiene `Estado`, `Fecha_creacion` ni `Creado_por`.** Tus otras tablas sí los tienen. Sin `Estado` no hay forma de desvincular una empresa sin borrarla.
- **Agregar un CHECK en `Tipo_persona`** con los valores `PJ` y `PN`.
- **Agregar `EMPRESA_VINCULADA` a `TIPO_EVENTO`** en tus inserts.

Esta va con el formato recortado que acordamos: sin la sección de flujo, la bitácora referida a la HU-13 y los pendientes al final, como tareas aparte.

---

## 🟢 HU-04 · Vincular productos de la empresa
**Microservicio:** `ms-grupos-economicos`
**Endpoints:** `GET /v1/grupos-economicos/{id}/empresas/{idEmpresa}/productos-core` (listar desde COBIS) · `POST /v1/grupos-economicos/{id}/empresas/{idEmpresa}/productos` (vincular)

**Como** empleado de Servicio de Canales, **quiero** ver los productos que la empresa tiene en COBIS y elegir cuáles entran a BELC, **para** que después se puedan agrupar y asignar a roles.

### Componentes
| Componente | Qué hace en esta historia |
|---|---|
| **BackOffice web** | Muestra los productos de la empresa en COBIS con casillas para elegir |
| **API Gateway** | Valida el token del empleado y enruta al microservicio |
| **ms-grupos-economicos** | Consulta COBIS, valida y guarda los productos elegidos |
| **Integración con COBIS** (o un mock al inicio) | Lista los productos del cliente empresa. **Solo lectura** |
| **Base BELC** | `Empresas_asociadas`, `Producto_vinculacion`, `Tipo_producto`, `moneda` y `BITACORA_EVENTO` |
| **Librería de bitácora** (HU-13) | Registra el evento `PRODUCTO_VINCULADO` por cada producto |

### Modelo de datos
**Se leen:**
- `Empresas_asociadas`: que la empresa exista y pertenezca al grupo de la URL, y su `core_empresasoc_id` para consultar COBIS.
- `Tipo_producto` y `moneda`: para convertir los códigos de COBIS a los ids de tus catálogos.

**`Producto_vinculacion`** (una fila por producto elegido)
| Campo | De dónde viene |
|---|---|
| `id_producto_vinculado` | Lo genera la base (PK) |
| `id_empresa_asociada` | De la URL |
| `Core_module`, `Core_product_id`, `Core_cliente_id` | De COBIS |
| `id_tipo_producto` | Código de COBIS → `Tipo_producto` |
| `id_moneda` | Código de COBIS → `moneda` |
| `Numero_mascara` | Lo arma el microservicio, por ejemplo `****4521`. **Nunca se guarda el número completo** |
| `Estado_core` | El estado que trae COBIS |
| `Estado_Vinculacion` | `ACTIVO` |
| `Fecha_vinculacion`, `Fecha_ultima_sincronizacion` | Fecha del servidor |
| `Creado_por` | El empleado |

### Contrato
**1. Listar desde COBIS:** `GET .../empresas/201/productos-core`
```json
{
  "productos": [
    { "coreModule": "CTE", "coreProductId": "PRD-778120", "tipoProducto": "CUENTA_CORRIENTE", "moneda": "USD", "numeroMascara": "****4521", "estadoCore": "ACTIVA", "yaVinculado": false },
    { "coreModule": "DPF", "coreProductId": "PRD-778131", "tipoProducto": "DEPOSITO_PLAZO_FIJO", "moneda": "USD", "numeroMascara": "****0031", "estadoCore": "ACTIVA", "yaVinculado": true }
  ]
}
```
**2. Vincular:** `POST .../empresas/201/productos`
```json
{ "productos": ["PRD-778120"] }
```
**Respuesta 201**
```json
{ "vinculados": [ { "idProductoVinculado": 9001, "etiqueta": "Cuenta corriente ****4521" } ] }
```
**Errores:** `403` sin permiso · `404` el grupo o la empresa no existe, o la empresa no es de ese grupo · `409` el producto ya está vinculado · `422` el producto no es de esa empresa en COBIS, o su tipo no existe en `Tipo_producto` · `504` COBIS no respondió a tiempo.

### Criterios de aceptación
1. Solo se vinculan productos que COBIS devuelve para esa empresa. Al confirmar, **se vuelve a consultar COBIS**; no se confía en lo que manda la pantalla.
2. Se guarda solo el número enmascarado; el número completo nunca llega a la base.
3. Un producto no se vincula dos veces (409).
4. Si se eligen varios productos y uno falla, no se guarda ninguno.
5. Cada producto queda con `Estado_Vinculacion = ACTIVO`, su `Estado_core` y la fecha de sincronización.
6. Si COBIS no responde, devuelve 504 y no guarda nada.
7. Cada producto vinculado registra el evento `PRODUCTO_VINCULADO` (HU-13).

---

### Tareas aparte (cambios a tu DDL)
- **Agregar un UNIQUE en `(Core_module, Core_product_id)`** en `Producto_vinculacion`. Sin él, el criterio 3 depende solo del código.
- **`Producto_vinculacion` tiene `Moneda` y también `id_moneda`.** Deja solo `id_moneda`, o se pueden contradecir.
- **Agregar un CHECK en `Estado_Vinculacion`** con los estados de tu diapositiva 6: `PENDIENTE`, `ACTIVO`, `SUSPENDIDO`, `DESVINCULADO`.
- **Agregar `PRODUCTO_VINCULADO` a `TIPO_EVENTO`.**

### Dependencias
- **Equipo del core:** qué servicio de COBIS lista los productos de un cliente empresa y qué códigos usa para módulo, tipo de producto y moneda, para cargar la equivalencia en tus catálogos.
- **Una pregunta para ti:** `Core_cliente_id` en `Producto_vinculacion`, ¿es el mismo valor que `core_empresasoc_id` en `Empresas_asociadas`? Si es el mismo, sobra en `Producto_vinculacion`, porque ya se obtiene a través de la empresa.

**Fuera de alcance:** desvincular productos y la sincronización periódica de `Estado_core` con COBIS. Esa sincronización merece su propia historia.


Va como historia técnica: componentes, tablas, consulta SQL, contrato y criterios verificables. Usé el formato de respuesta de tu contrato `GET .../usuarios`.

---

## 🟢 HU-07 · Listar usuarios del grupo económico
**Microservicio:** `ms-autogestion` · **Endpoint:** `GET /v1/grupos-economicos/{id}/usuarios?pagina=1&tamanoPagina=20`

**Como** administrador del grupo económico, **quiero** ver a los usuarios del grupo con su rol y estado, incluidas las invitaciones pendientes, **para** gestionar quién tiene acceso.

### Componentes
| Componente | Qué hace |
|---|---|
| **Canal BELC** (web y app) | Pantalla del paso 3 de la autogestión |
| **API Gateway** | Valida el token de sesión: audiencia y `belc_enabled` |
| **ms-autogestion** | Valida el permiso, consulta la base y arma la respuesta paginada |
| **Base BELC** | Las tablas de abajo, **solo lectura** |
| ForgeRock, COBIS | **No intervienen.** El nombre sale de `Nombre_mostrar` |

### Tablas (solo lectura)
| Tabla | Campos que usa | Para qué |
|---|---|---|
| `Usuario_grupo_economico` | `Identity_id`, `id_role`, `Estado`, `id_grupo_economico` | Los usuarios que ya aceptaron |
| `Belc_persona_canal` | `Nombre_mostrar` | El nombre que se muestra |
| `role` | `Nombre_role` | El nombre del rol |
| `Grupo_economico_administrador` | `Identity_id`, `id_grupo_economico`, `Estado` | El indicador `esAdministrador` |
| `afiliacion` | `id_afiliacion`, `id_rol_asignado`, `Estado`, `Fecha_expiracion` | Las invitaciones que todavía no se aceptan |

### Consulta
```sql
SELECT * FROM (
  -- Usuarios que ya aceptaron
  SELECT u.Identity_id          AS identityId,
         NULL                   AS idAfiliacion,
         p.Nombre_mostrar       AS nombre,
         r.Nombre_role          AS rol,
         u.Estado               AS estado,
         NULL                   AS invitacionVence,
         (ga.Identity_id IS NOT NULL) AS esAdministrador
    FROM Usuario_grupo_economico u
    JOIN Belc_persona_canal p ON p.Identity_id = u.Identity_id
    JOIN role r               ON r.id_role     = u.id_role
    LEFT JOIN Grupo_economico_administrador ga
           ON ga.Identity_id = u.Identity_id
          AND ga.id_grupo_economico = u.id_grupo_economico
          AND ga.Estado = 'ACTIVO'
   WHERE u.id_grupo_economico = :id
  UNION ALL
  -- Invitaciones pendientes
  SELECT NULL, a.id_afiliacion, a.Nombre, r.Nombre_role,
         CASE WHEN a.Fecha_expiracion < NOW() THEN 'INVITACION_VENCIDA'
              ELSE 'INVITACION_ENVIADA' END,
         a.Fecha_expiracion, FALSE
    FROM afiliacion a
    JOIN role r ON r.id_role = a.id_rol_asignado
   WHERE a.id_grupo_economico = :id
     AND a.Estado = 'PENDIENTE'
) t
ORDER BY nombre
LIMIT :tamanoPagina OFFSET :offset;
```
El total de usuarios para la paginación se obtiene con la misma consulta usando `COUNT(*)`.

### Contrato
**Respuesta 200**
```json
{
  "usuarios": [
    { "identityId": "3f2a9c1e-…", "idAfiliacion": null, "nombre": "Juan Pérez", "rol": "Rol Creador administrativo Empresa 1", "estado": "ACTIVO", "invitacionVence": null, "esAdministrador": true },
    { "identityId": null, "idAfiliacion": 77, "nombre": "Karina Pérez", "rol": "Rol Creador monetario Empresa 1", "estado": "INVITACION_ENVIADA", "invitacionVence": "2026-10-17T14:05:12-05:00", "esAdministrador": false }
  ],
  "paginacion": { "pagina": 1, "tamanoPagina": 20, "totalUsuarios": 7 }
}
```
**Parámetros:** `pagina` (por defecto 1) y `tamanoPagina` (por defecto 20, máximo 100).
**Errores:** `400` parámetros inválidos · `401` sin sesión · `403` sin permiso o no pertenece al grupo.

### Criterios de aceptación
1. Devuelve los usuarios del grupo con su nombre, rol, estado e indicador de administrador.
2. Incluye las invitaciones pendientes con su fecha de vencimiento. Las vencidas aparecen como `INVITACION_VENCIDA`, para que el administrador pueda reenviarlas.
3. Una invitación aceptada no aparece dos veces: deja de estar `PENDIENTE` y la persona se muestra como usuario.
4. Está paginado; un `tamanoPagina` mayor que 100 responde 400.
5. El nombre sale de `Nombre_mostrar`; no se hace ninguna llamada a ForgeRock.
6. **Solo se ven los usuarios de un grupo al que pertenece quien consulta.** Pedir `/grupos-economicos/20/usuarios` siendo del grupo 10 responde 403, aunque tenga el permiso.
7. Requiere el permiso `CREAR_USUARIOS`. Los 403 se registran en la bitácora (HU-13); las consultas exitosas no.
8. Con 1000 usuarios en el grupo, responde en menos de 1 segundo.

### Tareas aparte (cambios a tu DDL)
- **Agregar `Nombre` a `afiliacion`.** Tu propia nota del contrato lo dice: sin esa columna no se puede mostrar a Karina antes de que acepte.
- **Agregar índices:** `Usuario_grupo_economico (id_grupo_economico, Estado)` y `afiliacion (id_grupo_economico, Estado)`, para cumplir el criterio 8.

### Dependencias
- **HU 2 (resolver la persona en cada operación):** de ella salen la validación del permiso y la de pertenencia al grupo.
- **La PO:** si el permiso para ver usuarios es `CREAR_USUARIOS` u otro de solo consulta.
HU-10 · Consultar productos con saldo

Microservicio: ms-productos · Endpoint: GET /v1/grupos-economicos/{id}/productos

Como usuario con permiso de consulta, quiero ver los productos de mis grupos de productos con su saldo disponible, para conocer la posición de las empresas del grupo.

Componentes
Componente	Qué hace
Canal BELC (web y app)	Pantalla de productos, agrupada por empresa
API Gateway	Valida el token de sesión: audiencia y belc_enabled
ms-productos	Resuelve qué productos puede ver el usuario, pide los saldos a COBIS y arma la respuesta
Base BELC	Las tablas de abajo, solo lectura
Integración con COBIS (o un mock al inicio)	Devuelve el saldo disponible de cada producto. Solo lectura
ForgeRock	No interviene
Tablas (solo lectura)
Tabla	Campos que usa	Para qué
Usuario_grupo_economico	Identity_id, id_role, id_tipo_visualizacion, Estado	El rol del usuario en el grupo
Rol_grupo_producto	id_role, id_grupo_producto	Qué grupos de productos tiene el rol
Grupo_producto	id_Grupo_producto, Estado	Solo los grupos activos
Grupo_producto_detalle	id_grupo_producto, id_producto_vinculacion	Qué productos hay en cada grupo
Producto_vinculacion	Core_module, Core_product_id, Numero_mascara, id_tipo_producto, id_moneda, Estado_Vinculacion	Los datos del producto y su llave en COBIS
Empresas_asociadas	Nombre, Tipo_persona	Agrupar los productos por empresa
Tipo_producto, moneda	Nombre, Codigo	La etiqueta y la moneda
Tipo_visualizacion	Codigo	Mixto o Separado
Consulta
sql
SELECT DISTINCT
       pv.id_producto_vinculado, pv.Core_module, pv.Core_product_id, pv.Numero_mascara,
       tp.Codigo AS tipoProducto, tp.Nombre AS nombreTipoProducto, m.Codigo AS moneda,
       ea.id_empresa_asociada, ea.Nombre AS empresa, ea.Tipo_persona
  FROM Usuario_grupo_economico u
  JOIN Rol_grupo_producto rgp     ON rgp.id_role = u.id_role
  JOIN Grupo_producto gp          ON gp.id_Grupo_producto = rgp.id_grupo_producto
                                 AND gp.Estado = 'ACTIVO'
  JOIN Grupo_producto_detalle gpd ON gpd.id_grupo_producto = gp.id_Grupo_producto
  JOIN Producto_vinculacion pv    ON pv.id_producto_vinculado = gpd.id_producto_vinculacion
                                 AND pv.Estado_Vinculacion = 'ACTIVO'
  JOIN Empresas_asociadas ea      ON ea.id_empresa_asociada = pv.id_empresa_asociada
  JOIN Tipo_producto tp           ON tp.id_tipo_producto = pv.id_tipo_producto
  JOIN moneda m                   ON m.id_moneda = pv.id_moneda
 WHERE u.Identity_id = :identityId
   AND u.id_grupo_economico = :id
   AND u.Estado = 'ACTIVO'
 ORDER BY ea.Nombre, tp.Nombre;

Con el resultado, el microservicio pide en una sola llamada los saldos a COBIS, usando la lista de Core_module y Core_product_id.

Contrato

Respuesta 200 (tu contrato)

json
{
  "idGrupoEconomico": 10,
  "tipoVisualizacion": "MIXTO",
  "fechaActualizacion": "2026-10-14T14:05:12-05:00",
  "empresas": [
    {
      "idEmpresaAsociada": 101, "nombre": "Empresa 1", "tipoPersona": "PJ",
      "productos": [
        { "idProductoVinculado": 9001, "tipoProducto": "CUENTA", "nombreTipoProducto": "Cuenta corriente",
          "numeroMascara": "****4521", "moneda": "USD", "saldoDisponible": 125430.75, "estado": "ACTIVA" }
      ]
    }
  ],
  "acciones": ["VER_DETALLE", "DESCARGAR_ESTADO_CUENTA", "DESCARGAR_EXTRACTO"]
}

Errores: 401 sin sesión · 403 sin permiso o no pertenece al grupo · 502 COBIS respondió con error · 504 COBIS no respondió a tiempo (retryable: true).

Criterios de aceptación
Solo muestra productos de los grupos de productos activos del rol, y solo productos con Estado_Vinculacion = ACTIVO.
Si un producto está en dos grupos de productos del mismo rol, aparece una sola vez.
El saldo viene de COBIS en línea, en una sola llamada para todos los productos. fechaActualizacion es la hora de la respuesta de COBIS.
Los saldos no se guardan ni en la base ni en los logs.
Si COBIS no responde dentro del tiempo límite, devuelve 504 con retryable: true y el formato de error de tu contrato.
Sin el permiso CONSULTA_PRODUCTOS_DETALLE, responde 403. Es el caso de Juan en tu contrato, que no tiene permisos monetarios.
Pedir los productos de un grupo al que no pertenece responde 403.
acciones incluye solo lo que el rol permite. Por ejemplo, DESCARGAR_ESTADO_CUENTA aparece solo si tiene ese permiso.
Tareas aparte
Índices: Rol_grupo_producto (id_role) y Grupo_producto_detalle (id_grupo_producto).
Dependencias y decisiones
HU 2: de ella salen la validación de la sesión, el permiso y la pertenencia al grupo.
Equipo del core: el servicio de COBIS que consulta saldos de varios productos en una llamada, y su tiempo de respuesta para fijar el tiempo límite.
La PO: Mixto o Separado. Cambia cómo la pantalla agrupa los productos; el microservicio solo devuelve el valor.
Una decisión para ti: si COBIS responde los saldos de algunos productos y de otros no, ¿se responde 504 para todo o se devuelven los productos con saldoDisponible: null? Tu contrato hoy solo contempla el 504 completo. La segunda opción es más útil para el usuario, pero hay que agregarla al contrato.


### Pendientes con otros equipos
- **Equipo del core:** qué servicio de COBIS se usa para buscar empresas, por qué identificador (RUC o número de cliente) y cuál es su tiempo de respuesta, para fijar el timeout.
- **Líder técnico:** cómo se autentica el empleado del banco (igual que en la HU-02).

**Fuera de alcance:** desvincular una empresa o moverla de grupo, y vincular sus productos (eso es la HU-04)


🟢 HU-11 · Consultar detalle de producto

Microservicio: ms-productos · Endpoint: GET /v1/grupos-economicos/{id}/productos/{idProductoVinculado}

Como usuario con el permiso CONSULTA_PRODUCTOS_DETALLE, quiero ver el detalle de un producto de mis grupos de productos, para conocer sus saldos y condiciones.

Componentes
Componente	Qué hace
Canal BELC (web y app)	Pantalla de detalle; se abre desde la lista de la HU-10
API Gateway	Valida el token de sesión: audiencia y belc_enabled
ms-productos	Verifica que el producto sea visible para el rol, pide el detalle a COBIS y arma la respuesta
Base BELC	Las mismas tablas de la HU-10, solo lectura
Integración con COBIS (o un mock al inicio)	Devuelve el detalle del producto. Solo lectura
ForgeRock	No interviene
Tablas (solo lectura)

Las mismas de la HU-10: Usuario_grupo_economico, Rol_grupo_producto, Grupo_producto, Grupo_producto_detalle, Producto_vinculacion, Empresas_asociadas, Tipo_producto y moneda. Aquí se usan para un solo producto.

Consulta (autorización y llave en COBIS)
sql
SELECT pv.id_producto_vinculado, pv.Core_module, pv.Core_product_id, pv.Numero_mascara,
       tp.Codigo AS tipoProducto, tp.Nombre AS nombreTipoProducto, m.Codigo AS moneda,
       ea.Nombre AS empresa
  FROM Producto_vinculacion pv
  JOIN Empresas_asociadas ea ON ea.id_empresa_asociada = pv.id_empresa_asociada
  JOIN Tipo_producto tp      ON tp.id_tipo_producto     = pv.id_tipo_producto
  JOIN moneda m              ON m.id_moneda             = pv.id_moneda
 WHERE pv.id_producto_vinculado = :idProductoVinculado
   AND pv.Estado_Vinculacion = 'ACTIVO'
   AND EXISTS (
       SELECT 1
         FROM Usuario_grupo_economico u
         JOIN Rol_grupo_producto rgp     ON rgp.id_role = u.id_role
         JOIN Grupo_producto gp          ON gp.id_Grupo_producto = rgp.id_grupo_producto
                                        AND gp.Estado = 'ACTIVO'
         JOIN Grupo_producto_detalle gpd ON gpd.id_grupo_producto = gp.id_Grupo_producto
        WHERE u.Identity_id = :identityId
          AND u.id_grupo_economico = :id
          AND u.Estado = 'ACTIVO'
          AND gpd.id_producto_vinculacion = pv.id_producto_vinculado);

Si la consulta no devuelve nada, no se llama a COBIS.

Contrato

Respuesta 200 (ejemplo para una cuenta corriente)

json
{
  "idProductoVinculado": 9001,
  "empresa": "Empresa 1",
  "tipoProducto": "CUENTA",
  "nombreTipoProducto": "Cuenta corriente",
  "numeroMascara": "****4521",
  "moneda": "USD",
  "estado": "ACTIVA",
  "saldos": {
    "disponible": 125430.75,
    "contable": 127930.75,
    "retenido": 2500.00,
    "enTransito": 0.00
  },
  "fechaApertura": "2019-03-12",
  "fechaActualizacion": "2026-10-27T09:15:40-05:00",
  "acciones": ["DESCARGAR_ESTADO_CUENTA", "DESCARGAR_EXTRACTO"]
}

Los campos de saldos y las fechas son de ejemplo. Cada tipo de producto trae otros datos; un plazo fijo, por ejemplo, tendría tasa, plazo y fecha de vencimiento.

Errores: 401 sin sesión · 403 sin permiso, o el producto no está en los grupos de productos del rol · 404 el producto no existe o está desvinculado · 502 COBIS respondió con error · 504 COBIS no respondió a tiempo (retryable: true).

Criterios de aceptación
Si el producto no está en los grupos de productos activos del rol, responde 403 sin llamar a COBIS.
Los datos del detalle vienen de COBIS en línea. BELC solo agrega el nombre de la empresa y la etiqueta.
Nunca muestra el número de cuenta completo, solo numeroMascara. Si COBIS devuelve el número completo, el microservicio lo descarta.
Un producto con Estado_Vinculacion distinto de ACTIVO responde 404.
Sin el permiso CONSULTA_PRODUCTOS_DETALLE, responde 403.
El detalle no se guarda ni en la base ni en los logs.
Si COBIS no responde dentro del tiempo límite, devuelve 504 con retryable: true.
acciones incluye solo lo que el rol permite.
Dependencias y decisiones
HU 2 y HU-10: reutiliza la validación de sesión y la consulta de visibilidad.
La PO: qué campos se muestran en el detalle de cada tipo de producto (cuenta corriente, ahorro, plazo fijo, tarjeta). Sin eso, el contrato es solo un ejemplo.
Equipo del core: qué servicio de COBIS da el detalle y qué campos devuelve por tipo de producto.
Una decisión para ti: ¿403 o 404 cuando el producto no es visible para el usuario? El 404 no revela que el producto existe, y es lo más seguro. Lo dejé en 403 porque así lo escribiste; tú decides.

🟢 HU-13 · Registrar eventos en la bitácora

Componente: librería compartida que usan todos los microservicios. No es un endpoint.

Como microservicio BELC, quiero registrar cada operación relevante en BITACORA_EVENTO, para tener una auditoría completa e inalterable desde el primer día.

Componentes
Componente	Qué hace
Librería de bitácora	Expone un método para registrar eventos y llena sola los datos de la petición
Todos los microservicios BELC	Llaman a la librería en cada escritura y en cada acceso denegado
Base BELC	BITACORA_EVENTO (solo inserción) y TIPO_EVENTO (solo lectura)
Usuario de base de datos de la aplicación	Tiene solo permiso de INSERT sobre la bitácora
Tablas

TIPO_EVENTO (solo lectura): Codigo, Categoria, Criticidad, Visible_cliente, Estado.

BITACORA_EVENTO (solo inserción). Quién llena cada campo:

Lo llena la librería sola (a partir de la petición)	Lo pasa el microservicio
Fecha_evento_utc, Canal, Identity_id o Usuario_backoffice, Direccion_ip, Dispositivo, id_sesion, id_correlacion, Metodo_http, Uri, id_grupo_economico, Hash_registro	id_tipo_evento (por su código), Resultado, Entidad, id_registro, Valor_anterior, Valor_nuevo, Codigo_error, Detalle y, si es una transacción, Monto, Moneda, Cuenta_origen, Cuenta_destino, Tipo_movimiento
Cómo se usa (ejemplo)
java
bitacora.registrar(
    "EMPRESA_VINCULADA",          // código en TIPO_EVENTO
    Resultado.EXITO,
    "Empresas_asociadas", "201",  // entidad e id del registro
    null,                         // valor anterior
    empresaJson                   // valor nuevo
);

El lenguaje del ejemplo es Java; ajústalo al que use el equipo.

Criterios de aceptación
Solo inserta. El usuario de base de datos de la aplicación tiene únicamente GRANT INSERT sobre BITACORA_EVENTO. Un UPDATE o DELETE falla por permisos, no solo porque el código no lo hace.
Cada evento guarda tipo, resultado, canal, usuario, id_correlacion y Fecha_evento_utc con milisegundos.
En una operación exitosa, la bitácora se escribe en la misma transacción que el cambio. Si la bitácora falla, el cambio se deshace.
En una operación fallida (403, 504, validación), la bitácora se escribe en su propia transacción. Si no, al deshacerse la operación se perdería el registro del fallo.
id_correlacion es el mismo correlationId que se devuelve en las respuestas de error.
Nunca guarda datos sensibles: ni tokens, ni contraseñas, ni OTP, ni números de cuenta completos. Cuenta_origen y Cuenta_destino van enmascaradas, y los JSON de Valor_anterior y Valor_nuevo excluyen esos campos.
Hash_registro es el SHA-256 del contenido del registro, para detectar si alguien lo altera.
Un código de evento que no existe o está inactivo en TIPO_EVENTO lanza un error que las pruebas detectan, para que nunca llegue a producción.
Tareas aparte (cambios a tu DDL)
Convertir la bitácora a sintaxis de MySQL (la tabla de arriba).
Normalizar los valores a mayúsculas: Resultado usa 'Exito' y 'Fallo', y TIPO_EVENTO.Estado usa 'Activo'. Pásalos a EXITO, FALLO y ACTIVO, con COLLATE utf8mb4_bin, como el resto de tus estados. Es justo el punto 3 de la revisión de Domingo.
Separar id_aprobacion en dos columnas, administrativa y transaccional, como dice tu presentación del DDL.
Cargar los códigos de evento en TIPO_EVENTO: GRUPO_ECONOMICO_CREADO, EMPRESA_VINCULADA, PRODUCTO_VINCULADO, ACCESO_DENEGADO, y los que vayan saliendo en cada historia.
Dependencias
DBA: crear el usuario de la aplicación con solo INSERT sobre la bitácora.
DBA y Seguridad: Activity Streams y S3 Object Lock, para que la bitácora tampoco se pueda alterar desde fuera de la aplicación. Ya está en tus pendientes técnicos.
