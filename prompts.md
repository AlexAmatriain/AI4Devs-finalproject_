# prompts.md · Registro del uso de IA

Este documento resume cómo se usó la IA para construir el MVP de Prosolia. Para cada apartado del índice indica la herramienta, los prompts concretos que se lanzaron y los ajustes humanos posteriores. No recoge todos los prompts, solo los que mejor muestran el método de trabajo.

> **Origen de los prompts.** Cada prompt se ha reconstruido a partir de la propuesta OpenSpec que generó (`backend/openspec/changes/archive/<fecha>-<cambio>/proposal.md` y su equivalente en `frontend/`). Por eso se indica el cambio y, cuando existe, el commit. El único texto literal es el de `backend/prompt.txt` (apartado 3.4).

### Herramientas y modelos

| Herramienta / modelo | Uso |
|---|---|
| **Claude Code** (CLI, Windows) | Agente principal en `backend/` y `frontend/`. Se encargó de leer el repositorio, redactar especificaciones, implementar, generar migraciones de EF Core, compilar y preparar commits. |
| **OpenSpec** | Flujo *spec-driven* con `/opsx:explore`, `/opsx:propose`, `/opsx:apply` y `/opsx:archive`. Se archivaron 83 cambios en backend y 35 en frontend. |
| **`AGENTS.md`** | Contexto fijo en la raíz, `backend/` y `frontend/`: objetivo, stack, arquitectura hexagonal y la regla de redactar las specs en castellano. |
| **Postman** | Colecciones reales (`FusionSolar.postman_collection.json`, `Prosolia_Streem_*.postman_collection.json`) que se usaron como referencia de las APIs en los prompts. |
| **Claude Sonnet 5** | Bloque de informes mensuales (`2f26f69`, `24982d5`, `0cb47f8`, `8c92344`, `b0c44f6`). |
| **Claude Sonnet 5.5** | Correcciones de informes y de código de planta (`98e3ab6`, `e6c8b76`, `d131e21`). |
| **Claude Opus 5.5** | Documentación de entrega en `docs_v2/` y este registro. |

## Índice

1. [Descripción general del producto](#1-descripción-general-del-producto)
2. [Arquitectura del sistema](#2-arquitectura-del-sistema)
3. [Modelo de datos](#3-modelo-de-datos)
4. [Especificación de la API](#4-especificación-de-la-api)
5. [Historias de usuario](#5-historias-de-usuario)
6. [Tickets de trabajo](#6-tickets-de-trabajo)
7. [Pull requests](#7-pull-requests)

---

## 1. Descripción general del producto

### 1.1 Contexto inicial escrito a mano (`AGENTS.md`)

El objetivo y el alcance no se pidieron a la IA: se escribieron antes del primer prompt y se cargan en todas las sesiones.

```text
## OBJETIVO
Construir un producto minimo viable donde se construira y mostrara informacion
util sobre intervenciones necesarias en plantas solares a partir de la ingesta
de informacion cruda de APIS de fabricantes de dispositivos industriales,
eliminando ruido y añadiendo conocimiento propio de los desarrolladores.
```

### 1.2 Reglas de negocio de las intervenciones

Cambio `2026-09-03-upsert-intervenciones-alarmas-ac`:

```text
/opsx:propose Cuando ya exista una intervención para la misma alarma AC
(id_alarma_ac), actualízala en lugar de ignorarla. Recorre todas las alarmas
AC en cada ejecución. Al crear o actualizar:
- fecha_activacion_utc = ts_inicio de la alarma AC
- fecha_desactivacion_utc = ts_fin (null si sigue abierta)
- fecha_plan_utc, solo si el tipo es CORRINM o CORRPLAN:
  URGENTE → hoy; MAYOR → activación + 15 días; MENOR → activación + 30 días
- id_estado: solo plan → PLANIFICADA; plan + activación → ABIERTA;
  plan + activación + desactivación → CERRADA. En otro caso no lo toques.
Alinea el enum EstadoIntervencion con los ids reales de ESTADOS_INTERVENCION
(Abierta=1, Cerrada=5, Planificada=6, Cancelada=7).
```

### 1.3 Documento funcional del cliente → cambios coordinados

Cambios `2026-10-22-*` en backend y frontend (commit `48ccd26`):

```text
Lee CONTROL_DE_ESTADOS_PROSOLIA_v2.docx (en la raíz del repo) y conviértelo en
cambios OpenSpec, con el mismo nombre en backend y frontend cuando afecten a
ambos. El estado ya no se elige a mano: se calcula a partir de las fechas según
la tabla 3.2 del documento. Implementa las validaciones V-01 a V-08 devolviendo
el código de regla en el 400 para que el frontend traduzca el mensaje.
La cancelación pasa a ser un indicador con motivo obligatorio. Añade las
intervenciones en espera, la planificación vencida y el histórico de cambios.
```

```text
Genera CONTROL_DE_ESTADOS_v2_CAMBIOS_Y_PRUEBAS.md con dos partes: los cambios
funcionales explicados para un usuario del panel y una batería de pruebas
manuales (CP-xx) para validarlos tras el despliegue.
```

Resultado: 8 cambios (`estados-calculados-por-fechas`, `fechas-automaticas-editables`, `historico-intervenciones`, `intervenciones-en-espera`, `planificacion-vencida`, `recalculo-estados-existentes`, `tecnicos-desplegable-actuaciones` y `optimizar-generacion-alarmas-ac`) y el documento de pruebas.

```

### Ajustes humanos
- La IA añadió un filtro de "alarma activa" que nadie había pedido. Se eliminó en `450663d` y `quitar-filtro-alarma-activa`.
- La regla "Planificada + activación = Abierta" de la primera versión la cambió el cliente a "= Planificada" en el control de estados v2.
- Peticiones del cliente que se trasladaron a prompts nuevos: el histórico de alarmas Huawei desde junio (`6425abd`), el PR garantizado de PVsyst (`d131e21`) y los cambios del catálogo de sistemas (`d1c8d48`).

---

## 2. Arquitectura del sistema

### 2.1 Estructura hexagonal

Partió de `backend/AGENTS.md` y del cambio `2026-08-28-sync-plantas-fusionsolar`:

```text
/opsx:propose Implementa la sincronización de plantas desde FusionSolar:
login con userName + systemCode obteniendo el XSRF-TOKEN en FusionSolarClient,
llamada al listado de estaciones y upsert por Codigo en PLANTAS.
Define un puerto de dominio propio para el "origen de plantas", separado de
IRawDataSourceClient, que es para datos crudos. Expón
POST /api/plantas/sincronizar con un resumen creadas/actualizadas/errores.
Usa como referencia FusionSolar.postman_collection.json.
```

Este cambio fijó el patrón que siguieron los demás: puerto en `Prosolia.Domain/Ports`, caso de uso en `Prosolia.Application`, adaptador en `Prosolia.Infrastructure/ExternalApis` y endpoint mínimo en `Prosolia.Api/Endpoints`.

### 2.2 Series temporales en InfluxDB

Cambio `2026-09-18-volcar-curva-carga-streem-influxdb`:

```text
/opsx:propose Nuevo endpoint que, para cada contador de CONTADORES y cada
magnitud de Streem (Active_Generation, Reactive_positive, Reactive_negative,
Active_Consumption, Voltage, Potential_Power), llame a
GET /v2/installations/{codigo}/load_curve con resolution=10m y guarde cada
punto en el bucket gw_streem de InfluxDB, con el código del contador como tag.
Rango por defecto: desde el inicio del día UTC hasta ahora. Un fallo en un par
(contador, magnitud) no debe parar el resto. ITimeSeriesStore.WritePointsAsync
tiene que recibir el bucket, porque los KPIs de inversores van a "fusionsolar".
```

### 2.3 Ejecución programada en ECS

Cambio `2026-09-29-add-modo-job-batch`, commit `2e472c4`:

```text
/opsx:propose El backend corre en ECS con escalado horizontal. Necesito ejecutar
periódicamente los 11 casos de uso del pipeline (sincronizaciones, KPIs,
vinculación, curvas Streem, ingestas de alarmas, generación de alarmas AC e
intervenciones) sin que cada réplica los duplique. No quiero Hangfire ni Quartz.
Propón un modo "dotnet Prosolia.Api.dll job <nombre> [--desde] [--hasta]" que
ejecute un único caso de uso y termine con código 0/1, para lanzarlo como ECS
Task one-off desde EventBridge Scheduler o Step Functions. Sin el argumento
"job", el servicio HTTP debe arrancar exactamente igual que hoy.
```

Resultado: `Prosolia.Api/Jobs/JobRunner.cs` con los jobs `sincronizacion-plantas-fusionsolar`, `ingesta-alarmas-huawei`, `generacion-alarmas-ac`, `generacion-intervenciones`, etc.

### 2.4 Diagnóstico de coste en producción

Cambio `2026-10-22-optimizar-generacion-alarmas-ac`, commit `091e998`:

```text
/opsx:explore El job generacion-alarmas-ac (cada 5 min, Step Functions sobre
task-prosolia-backend) generó el 20/10/2026 538,72 GB de transferencia regional
entre RDS y ECS (5,39 USD, el 95,5 % del cargo). Desde
ventana-ingesta-alarmas-huawei-desde-junio cada ingesta inserta una fila por
alarma Huawei activa. Revisa GetByIdPlataformaAsync y GetCoincidentesAsync y
explica por qué crece el volumen.
```

```text
/opsx:propose Haz la generación incremental con una marca de agua por fila de
ALARMAS_ORIGEN. Agrupa por clave de alarma en PostgreSQL y calcula el cierre
con agregados: MAX(timestamp_milisec_ingesta_utc) en Huawei y
MIN(fecha_cierre) en Streem. Añade un paso que revise las ALARMAS_AC con ts_fin
nulo. Mantén la deduplicación, la no reevaluación de cerradas y los recuentos.
```

### Ajustes humanos
- Jenkins, Dockerfile, ECS y el esquema de producción (`8314b94`, `30095c7`, `4929a36`, `6ff060a`) se configuraron y validaron a mano.
- El coste lo detectó el equipo en la factura de AWS. La IA explicó el patrón N² después de recibir las cifras.
- Se pidió que el JobRunner registrara el detalle de cada error y no solo el recuento (`9dd142c`).
- La frecuencia de la ingesta Huawei se bajó por errores `failCode 407` de la API (`6c9e1e1`).

---

## 3. Modelo de datos

### 3.1 Migración inicial (`20260827145437_InitialCreate`)

```text
Crea con EF Core (código primero) la migración inicial de PostgreSQL con
exactamente las tablas y relaciones de relacion_postgresql_prosolia.jpeg:
PLANTAS, INVERSORES, CONTADORES, ALARMAS_HUAWEI, ALARMAS_STREEM,
MASTER_ALARMAS_AC, ALARMAS_ORIGEN, ORIGENES, ALARMAS_AC, INTERVENCIONES y sus
catálogos. Esquema "piloto". Cadena de conexión en appsettings por entorno.
```

### 3.2 Spec base del modelo

Cambio `2026-08-28-documentar-modelo-datos-postgresql`:

```text
/opsx:propose Documenta como spec de OpenSpec el estado actual del modelo
relacional: las 15 tablas existentes y las reglas de integridad que aplica EF
Core (PK, FK, unicidad, cascade/restrict/set null). No cambies código ni
esquema; es una baseline para los specs futuros.
```

### 3.3 Catálogos sembrados desde un SQL heredado

Cambio `2026-09-02-seed-catalogos-alarmas-ac`:

```text
/opsx:propose Convierte insercion_datos.sql en una migración de datos de EF
Core adaptada al esquema vigente: ORIGENES (ALARMAS_HUAWEI, ALARMAS_STREEM),
MASTER_ALARMAS_HUAWEI ("Fallo del dispositivo"), MASTER_ALARMAS_STREEM
("Rendimiento muy insuficiente", "Parada de produccion"),
SEVERIDAD_INTERVENCION (URGENTE, MAYOR, MENOR, WARNING), TIPOS_INTERVENCION
(CORRINM, CORRPLAN, PREVE), MASTER_ALARMAS_AC y ALARMAS_ORIGEN. Usa
ON CONFLICT (codigo) DO NOTHING porque algunos códigos ya existen. Borra el
.sql al terminar.
```

### 3.4 Lógica de alarmas AC escrita por el equipo (`backend/prompt.txt`, literal)

```text
Comprobar si ya existe en ALARMAS_AC un registro con la misma combinación de:
- id_master_ac
- ts_inicio
- id_planta
Si NO existe: proceder con la inserción del registro candidato.
Si existe: aplicar el tratamiento específico según el origen.

Caso id_origen = 5 (ALARMAS_STREEM)
- Si al menos uno tiene fecha_cierre distinto de null: tomar ese valor,
  transformarlo a timestamp en milisegundos UTC y usarlo como ts_fin.
- Si todos tienen fecha_cierre = null: no actualizar.

Caso id_origen = 4 (ALARMAS_HUAWEI)
- Si TODOS los timestamp_milisec_ingesta_utc son anteriores a
  (timestamp_actual_ms − periodicidad): actualizar ts_fin = timestamp actual.
- En caso contrario: no realizar ninguna acción.
```

### 3.5 Nuevas entidades del dominio

Cambio `2026-09-29-add-actuaciones-intervenciones`, commit `fc99b31`:

```text
/opsx:propose Añade la entidad actuación (1 intervención → N actuaciones).
Obligatorios: inicio, fin y descripción. Opcionales: técnicos, repuestos e
imágenes. Reglas: R1+R2, el fin no supera fecha_desactivacion_utc de la
intervención; R3, fin ≥ inicio; R4, sin solapes entre actuaciones no anuladas
de la misma intervención. Anulación lógica y histórico por campo (usuario,
fecha, valor anterior y nuevo). Tablas ACTUACIONES, IMAGENES_ACTUACION e
HISTORICO_ACTUACIONES. El frontend irá en un cambio hermano.
```

### 3.6 Tabla del modelo para la entrega

```text
Documenta en docs_v2 el modelo de datos con nombres genéricos (ALARMAS_CRUDAS,
TIPOS_ALARMA, REGLAS_ALARMA, ALARMAS...): para cada tabla, sus campos con tipo
y descripción, las relaciones, las series temporales de InfluxDB y lo que
queda pendiente de definir.
```

### Ajustes humanos
- La regla de deduplicación y cierre (3.4) es el conocimiento propio del equipo. Se escribió a mano para que la IA no la infiriera.
- En `insercion_datos.sql` se corrigieron a mano las columnas que no existían (`descripcion` → `codigo`, `codigo` → `id_plataforma`).
- El modelo se ajustó con 47 migraciones pequeñas, por ejemplo `EliminarCamposPlantas`, `EliminarEstadoAlarmasAc` y `AnadirCodigoPlanta`, en lugar de rehacer el esquema.
- Las columnas de PR pasaron a `numeric(8,4)` después de que un dato anómalo abortara un lote real (`AmpliarPrecisionPrInformesMensuales`, `e6c8b76`).
- El cálculo de `Ereno` cambió tres veces al observar el comportamiento real de Streem: `corregir-ereno-active-generation-corrected` (`d6dcd72`), `calcular-ereno-sin-active-generation-corrected` y `add-volcado-energia-corregida-streem` (`9e71ad7`). Primero se usó el valor corregido, después se descartó porque llega con unos 3 días de retraso, y al final se movió a un endpoint aparte que sobrescribe el dato más tarde.

---

## 4. Especificación de la API

### 4.1 Ingesta de alarmas Huawei

Cambio `2026-08-31-ingesta-alarmas-huawei`:

```text
/opsx:propose Endpoint POST /api/alarmas-huawei/ingesta que llame a
POST {base_url}/thirdData/getAlarmList con lotes de máximo 100 códigos de
PLANTAS. Reintentos con backoff y re-login para los failCode
403/407/305/306/307. Un lote fallido se registra y no aborta el resto.
Mapeo: alarmId→id_plataforma_huawei, stationCode→id_planta_huawei,
devTypeId→tipo_dispositivo_huawei, esnCode→num_serie, lev→severidad,
raiseTime→fecha_hora_inicio_timestamp_milisec. Índice único para que los
rangos solapados no dupliquen.
```

### 4.2 Ingesta de alarmas Streem

Cambio `2026-09-01-ingesta-alarmas-streem`:

```text
/opsx:propose Endpoint POST /api/alarmas-streem/ingesta?desde=&hasta= (por
defecto, las últimas 24 h) que llame a
GET /v2/alerts?start_date=...&end_date=...&all=true. El código de contador sale
del prefijo de installation_name hasta el tercer carácter no alfanumérico y se
resuelve contra CONTADORES.codigo. type→id_plataforma_streem,
created_at→fecha_inicio, closed_at→fecha_cierre. Las alarmas sin contador
conocido se omiten y se informan en la respuesta.
```

### 4.3 Generación de alarmas AC

Cambio `2026-09-02-endpoint-procesamiento-alarmas-ac`:

```text
/opsx:propose POST /api/alarmas-ac/procesar: por cada MASTER_ALARMAS_AC,
resuelve con ALARMAS_ORIGEN y ORIGENES la tabla cruda y el maestro del origen,
toma el id_plataforma de MASTER_ALARMAS_HUAWEI/STREEM y crea una fila en
ALARMAS_AC por cada fila cruda que coincida. Elimina la columna "tipo" de
ALARMAS_AC.
```

### 4.4 Listado paginado y resumen

Cambio `2026-09-28-paginacion-servidor-intervenciones`, commit `fce89b6`:

```text
/opsx:propose GET /api/intervenciones con paginación en servidor: aplica
primero todos los filtros sobre el total y después pagina. Filtros nuevos
combinables con AND: idSistema, idEquipoTecnico, alarmaActivaFinalizada y q
(texto libre sin distinguir mayúsculas en planta, código y descripción de la
alarma). Orden estable desempatando por id. Añade
GET /api/intervenciones/resumen con totales globales por estado, urgentes
activas, cerradas en 7 días y finalizadas con alarma activa.
```

### 4.5 Autenticación

Cambio `2026-09-22-add-autenticacion-bearer-token`, commit `4b6ec52`:

```text
/opsx:propose Autenticación con usuario y contraseña (hash, nunca en claro) y
token bearer. POST /api/auth/login devuelve el token. Todos los endpoints pasan
a exigir el token (401 si falta o no es válido), salvo /health.
POST /api/auth/usuarios crea o actualiza usuarios protegido con un secreto
compartido por variable de entorno, distinto del secreto de firma del token.
Sin roles.
```

### 4.6 Contrato para el frontend

Cambio `2026-09-03-enriquecer-consulta-intervenciones`:

```text
/opsx:propose Amplía IntervencionDto para el panel: PlantaCodigo, PlantaNombre,
AlarmaCodigo y AlarmaDescripcion (desde MASTER_ALARMAS_AC, al que se añade un
Codigo único). InversoresAfectados debe pasar a una lista de {IdInversor,
Codigo}. Solo afecta a la lectura: no toques /generar ni el PUT.
```

### Resultado (rutas reales en `Prosolia.Api/Endpoints`)

| Grupo | Rutas |
|---|---|
| Auth | `POST /api/auth/login`, `POST /api/auth/usuarios` |
| Activos | `POST /api/plantas/sincronizar`, `POST /api/inversores/sincronizar`, `POST /api/inversores/kpis`, `POST /api/contadores/sincronizar`, `POST /api/contadores/vincular-plantas` |
| Ingesta | `POST /api/alarmas-huawei/ingesta`, `POST /api/alarmas-streem/ingesta`, `POST /api/contadores/streem/curvas-variables`, `POST /api/contadores/streem/curvas-energia-corregida` |
| Procesamiento | `POST /api/alarmas-ac/procesar`, `POST /api/intervenciones/generar` |
| Intervenciones | `GET /`, `GET /resumen`, `GET/PUT /{id}`, `POST /`, `POST /{id}/cancelar`, `POST /{id}/deshacer-cancelacion`, `POST/DELETE /{id}/espera`, `GET /{id}/historico` |
| Actuaciones | `/api/intervenciones/{id}/actuaciones` (CRUD, `/anular`, `/historico`, `/imagenes`) |
| Informes | `POST /api/informes-mensuales/generar`, `GET /api/informes-mensuales/{idPlanta}/{anio}/{mes}/excel`, `POST /api/previsiones-pvsyst/importar`, `POST /api/previsiones-pvsyst/recalcular` |

### Ajustes humanos
- La primera versión de `/api/alarmas-ac/procesar` duplicaba filas si se ejecutaba dos veces. La propuesta lo aceptaba como limitación y se corrigió después con `deduplicar-actualizar-alarmas-ac` y `evitar-duplicados-en-ejecucion-alarmas-ac` (`a6a21a5`).
- La ingesta Huawei se rehízo para seguir la documentación oficial de FusionSolar (`1d2bb8b`). Además se pidió dar de alta automáticamente los `alarmId` desconocidos con valores por defecto MENOR, CORRPLAN y 60m (`52fe554`, `alta-automatica-alarmas-huawei-desconocidas`).
- En `docs_v2/README.md` las rutas se simplificaron y anonimizaron a mano (`/api/procesos/...`) respecto a las reales.

---

## 5. Historias de usuario

Durante el desarrollo, cada historia llegaba a la IA como una petición funcional concreta. Estos son los prompts originales, ordenados por historia.

| HU | Prompt de origen (resumen) | Cambio |
|---|---|---|
| HU-01 Login | "Pantalla de login; sin sesión se muestra el login y el token se envía en todas las peticiones" | `frontend: add-pantalla-login` |
| HU-03 Sincronización de activos | "Sincroniza inversores desde FusionSolar con su huawei_dev_type_id y vincula los contadores Streem a sus plantas, devolviendo los no vinculados" | `sincronizacion-inversores-fusionsolar`, `vincular-contadores-plantas` |
| HU-06 Sin ruido | "Una misma alarma no puede crearse dos veces en la misma ejecución" | `evitar-duplicados-en-ejecucion-alarmas-ac` |
| HU-07 Cierre automático | "Una alarma AC ya cerrada no vuelve a reevaluarse" | `no-reevaluar-alarmas-ac-cerradas` (`b07bb15`) |
| HU-12 Edición | "Si el usuario cierra a mano una intervención, respeta ese estado aunque la alarma AC siga abierta" | `respetar-intervenciones-en-estado-final` (`dc720ce`) |
| HU-13 Preventivos | "Botón + para crear una intervención preventiva manual con planta, fecha plan y descripción; el equipo se autocompleta desde la planta" | `crear-intervencion-preventiva-manual`, `autocompletar-equipo-en-creacion-manual` |
| HU-14 Actuaciones | "Botón Añadir actuación en el detalle, con modal, tabla de actuaciones y visor de imágenes" | `frontend: add-actuaciones-en-detalle-intervencion` |
| HU-15 Móvil | ver prompt 5.1 | `frontend-responsivo-movil` (`26f0d97`) |

### 5.1 Uso en campo

Cambio `2026-09-25-frontend-responsivo-movil`:

```text
/opsx:propose Los técnicos usan el panel desde el móvil en planta. Define
puntos de corte (móvil ≤ 640 px, tableta ≤ 1024 px), sin scroll horizontal,
inputs ≥ 16 px para evitar el zoom de iOS y objetivos táctiles ≥ 40 px.
En móvil, la tabla de intervenciones pasa a tarjetas, el detalle ocupa toda
la pantalla con la botonera fija y el modal de actuación también. Solo
presentación: sin cambios de API.
```

### 5.2 Panel en francés

Cambio `2026-10-13-add-i18n-es-fr`, commit `769eab8`:

```text
/opsx:propose Traduce el frontend a castellano y francés. Saca todos los
literales visibles a un catálogo (literales.md) con un id descriptivo. Pon un
selector de idioma arriba a la derecha que cambie al momento sin perder
filtros ni ediciones y que se recuerde entre sesiones. Traduce estados,
severidades, tipos y sistemas, pero envía a la API el código original. Las
descripciones de alarma van por diccionario de código AC. No traduzcas los
datos que introduce el usuario.
```

### Ajustes humanos
- Los criterios de aceptación de HU-09 (urgente hoy, mayor +15 días, menor +30 días) y HU-15 ("sin hacer zoom") vienen de las reglas reales, no de la IA.
- El desplegable de técnicos (`f1c837b`) y el indicador de alarma activa en finalizadas surgieron del uso real del panel por el cliente.

---

## 6. Tickets de trabajo

En la práctica, **cada cambio OpenSpec funcionó como un ticket**. `proposal.md` era la descripción, `specs/` los criterios de aceptación y `tasks.md` las subtareas que `/opsx:apply` iba marcando. Los tickets T-01 a T-27 de `docs_v2` agrupan esos cambios.

### 6.1 Ciclo completo de un ticket (informes mensuales)

```text
/opsx:explore Necesitamos un informe mensual por planta con producción,
irradiación, PR y disponibilidad, real frente a previsto (PVsyst), exportable
a Excel. ¿Qué datos nos faltan hoy y cómo lo partirías en cambios
independientes?
```

La exploración propuso 3 cambios (`18c9f07`). Al aparecer la irradiación real horaria (I_real v8), se reescribió en 4 (`e6a159d`):

```text
/opsx:propose add-datos-tecnicos-planta: importación de potencia pico,
potencia nominal y demás datos técnicos de cada planta desde Excel.
/opsx:propose add-previsiones-pvsyst: importación de los 12 meses de
epvsyst, ipvsyst y pr_pvsyst por planta, con reimportación parcial por mes.
/opsx:propose add-irradiacion-real-horaria: importación horaria de I_real;
una celda vacía no sobrescribe y se puede reimportar parcialmente.
/opsx:propose add-generacion-informes-mensuales: POST
/api/informes-mensuales/generar y export Excel separado en
GET /{idPlanta}/{anio}/{mes}/excel.
```

```text
/opsx:apply add-datos-tecnicos-planta      → 2f26f69
/opsx:apply add-previsiones-pvsyst         → 24982d5
/opsx:apply add-irradiacion-real-horaria   → 0cb47f8
/opsx:apply add-generacion-informes-mensuales → 8c92344
/opsx:archive (los 4)                      → b0c44f6
```

### 6.2 Ticket correctivo por petición del cliente

Cambio `2026-10-19-add-correccion-pvsyst-garantizado`, commit `d131e21`:

```text
/opsx:propose El cliente quiere que la previsión use el PR anual garantizado.
Añade epvsyst_corr y pr_pvsyst_corr a PREVISIONES_PVSYST. Para cada planta con
sus 12 meses: PR anual = media de pr_pvsyst ponderada por ipvsyst;
PR garantizado = PR anual − 5; f = garantizado / anual; aplica f a epvsyst y
pr_pvsyst. Las plantas que no cumplan (menos de 12 meses, valores ≤ 0) quedan
a NULL con aviso. Recalcula tras cada importación y con
POST /api/previsiones-pvsyst/recalcular. El informe mensual debe usar los
campos corregidos y dejar de restar 5 puntos fijos.
```

### 6.3 Ticket de identificación de plantas

Cambios `2026-10-02-anadir-codigo-planta` y `clave-planta-solo-digitos-tras-fr` (`2b77c06`, `98e3ab6`):

```text
/opsx:propose Añade codigo_planta a PLANTAS como identificador del cliente.
Úsalo en la importación de datos técnicos, PVsyst e irradiación, y en el
informe mensual (JSON, Excel y errores). Un código que coincida con varias
plantas debe rechazarse con error.
```

## 7. Pull requests

**Herramienta:** Claude Code redactó los mensajes de commit y las descripciones de merge. El repositorio está en GitLab.

### 7.1 Commit generado por la IA

```text
Haz commit de add-generacion-informes-mensuales. Mensaje en castellano:
título en imperativo con el nombre del cambio, y en el cuerpo qué se añade
(endpoints, tablas, cálculo de PR y disponibilidad), qué no se ha podido
verificar en este entorno y las dependencias con los otros 3 cambios.
```

Los commits resultantes llevan `Co-Authored-By: Claude Sonnet 5` (o `5.5`) y dejan constancia de lo pendiente. Por ejemplo, en el bloque de datos técnicos: *"confirmar el nombre real del campo total_cap contra la instancia de producción de FusionSolar"*.

### 7.2 Rama de feature e integración

```text
Implementa en la rama feature/informes-mensuales-backend los 4 cambios
OpenSpec de informes, uno por commit, y archívalos en un commit final.
Después prepara el merge a main con un resumen de los cambios, las
migraciones nuevas y los pasos de verificación (importar PVsyst de una
planta, generar el informe de un mes, descargar el Excel).
```

| Merge | Contenido |
|---|---|
| `ebb9365` `feature/informes-mensuales-backend` → `main` | 4 cambios OpenSpec, 4 migraciones (`AnadirDatosTecnicosPlanta`, `AnadirPrevisionesPvsyst`, `AnadirIrradiacionRealHoraria`, `AnadirInformesMensuales`) y su archivado |
| `cab77ec` `actuaciones` → `main` | Actuaciones con imágenes e histórico (backend y frontend) |

### Ajustes humanos
- Con un equipo pequeño y despliegue continuo desde Jenkins, la mayor parte del trabajo se integró directamente en `main`. Las ramas se reservaron para bloques grandes.
- La revisión se hizo sobre la **propuesta OpenSpec** antes de implementar, no sobre el diff. Así los errores se corregían antes de escribir código.
- Los merges, la resolución de conflictos (`55e1364`, `c385865`, `f5c0b57`) y la decisión de desplegar los hizo siempre una persona.
- Hay commits manuales que corrigen desviaciones de la IA respecto a la especificación: `b598159` (*"Correcciones desviaciones respecto a especificación original"*) y `450663d` (*"Elimina filtro no solicitado"*).
