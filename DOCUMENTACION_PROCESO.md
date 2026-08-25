# Documentacion del proceso - LiberacionDictRETIE

## 1. Resumen ejecutivo

`LiberacionDictRETIE` es una automatizacion UiPath que valida dictamenes RETIE/RETILAP emitidos en el dia, consulta informacion financiera en NetSuite y determina si cada dictamen puede quedar liberado o debe quedar retenido por motivos de facturacion/pago. Al finalizar, actualiza el Excel maestro de seguimiento, genera un archivo de resultados y envia correos de notificacion al comercial correspondiente.

El proceso trabaja principalmente sobre la hoja `DOC. SEG. ANTIGUO` del archivo de control `Proceso Dictamenes Digital`, alojado en SharePoint/OneDrive.

## 2. Objetivo del proceso

Automatizar la revision financiera de dictamenes tecnicamente liberados, cruzando los lotes/cotizaciones con facturas y proformas en NetSuite para:

- Liberar dictamenes cuando las validaciones financieras lo permiten.
- Retener dictamenes cuando existen inconsistencias, saldos pendientes, facturas vencidas, facturas/proformas anuladas o ausencia de informacion suficiente.
- Registrar observaciones de la decision.
- Marcar banderas de notificacion para correos de liberados o retenidos.
- Notificar al comercial el estado final del dictamen.

## 3. Alcance funcional

El robot procesa unicamente registros que cumplen las siguientes condiciones iniciales:

- La fecha de emision corresponde al dia de ejecucion.
- El campo de lote/cotizacion no esta vacio.
- El estado del area tecnica contiene `LIBERADO`.

No procesa dictamenes sin lote/cotizacion, dictamenes de fechas diferentes al dia actual ni registros sin liberacion tecnica.

## 4. Fuentes de informacion

### 4.1 Excel de seguimiento

Archivo origen:

- SharePoint/OneDrive: archivo de proceso de dictamenes digitales.
- Plantilla/local: `Proceso Dictamenes Digital.xlsx`.
- Hoja usada: `DOC. SEG. ANTIGUO`.

Columnas principales usadas por el robot:

| Columna | Campo | Uso |
| --- | --- | --- |
| A | FECHA DE EMISION | Filtro por fecha del dia. |
| B | No. COTIZACION | Lote/cotizacion a validar. Puede contener varios valores separados por guion. |
| C | No. INSPECCION | Dato para notificacion. |
| D | PROYECTO | Dato para asunto y cuerpo de correo. |
| E | INSPECTOR | Dato para notificacion. |
| F-G | Consecutivos | Dato para notificacion de liberados. |
| J | ESTADO DICTAMEN AREA TECNICA | Debe contener `LIBERADO` para procesarse. |
| K | OBSERVACION tecnica | Dato para notificacion de retenidos. |
| Q | ESTADO DICTAMEN AREA FINANCIERA | Resultado financiero: `LIBERADO` o `RETENIDO`. |
| R | OBSERVACION financiera | Motivo de liberacion o retencion. |
| T | NOMBRE DEL COMERCIAL | En el flujo se usa como destinatario del correo. |
| U | CORREO COMERCIAL / enlace | Se usa en plantillas de correo como URL o informacion asociada. |
| V | FECHA NOTIFICACION | Control persistente para evitar correos repetidos. Si esta vacia y el estado financiero queda `LIBERADO` o `RETENIDO`, el robot envia correo y luego escribe fecha/hora. |
| AD | NOTIFICACION RETENIDOS | Bandera `SI`/`No` para notificar retenidos. |
| AE | NOTIFICACION LIBERADOS | Bandera `SI`/`No` para notificar liberados. |

Nota: el XAML usa indices de columna de `DataTable`. Por ejemplo, `CurrentRow(16)` corresponde a la columna Q y `CurrentRow(17)` a la columna R.

### 4.2 NetSuite

El robot consulta NetSuite por SuiteQL mediante el endpoint REST:

`https://5845631.suitetalk.api.netsuite.com/services/rest/query/v1/suiteql`

Tablas/registros consultados:

- `CUSTOMRECORD_BIT_REPORTE_CARTERA`
- `CUSTOMRECORD_BIT_REPORTE_PROFORMA`
- `transaction`
- `transactionLine`

Campos relevantes usados en reglas:

- Lote de factura/proforma.
- Numero de factura.
- Cliente/tercero.
- Centro de costo.
- Saldo pendiente.
- Dias vencidos.
- Documento con el que fue pagado.
- Nota/memo de la transaccion.
- Lote a nivel de linea de transaccion.

## 5. Flujo general del proceso

### 5.1 Preparar archivo

1. Descarga desde SharePoint/OneDrive el archivo de control.
2. Guarda la ruta local descargada en `RUTA_INFORME_COPIA`.
3. Abre el archivo en Excel.
4. Elimina la primera fila para ajustar encabezados/datos.
5. Calcula la fecha actual en formato `dia/mes/anio`, con cero inicial para meses menores o iguales a 9.
6. Filtra la hoja `DOC. SEG. ANTIGUO` por `FECHA DE EMISION` igual a la fecha actual.
7. Recorre las filas visibles y escribe el numero de fila real en una columna auxiliar, usada luego para actualizar el Excel original.
8. Lee la hoja filtrada en `dataTable_resultadosInforme`.
9. Filtra el `DataTable` dejando solo registros con lote/cotizacion no vacio.
10. Filtra el `DataTable` dejando solo registros cuyo estado tecnico contiene `LIBERADO`.
11. Filtra el `DataTable` dejando solo registros con `FECHA NOTIFICACION` vacia.
12. Cierra procesos de Excel.

### 5.2 Validar lotes en NetSuite

Para cada registro del `DataTable`:

Nota: en esta etapa ya no deben entrar registros con `FECHA NOTIFICACION` diligenciada. Si la columna V tiene fecha/hora, el dictamen se considera notificado y se descarta antes de consultar facturas/proformas en NetSuite.

1. Toma el valor de `No. COTIZACION`.
2. Elimina espacios y separa multiples lotes cuando vienen con guion (`-`).
3. Para cada lote, inicializa variables de control:
   - `es_anulado = False`
   - `es_igual_lote = False`
   - `RECHAZO_POR_PAGO = True`
   - `observacion = vacio`
4. Consulta facturas en NetSuite asociadas al lote, memo o segmento de linea.
5. Si no encuentra facturas, consulta proformas asociadas al lote/orden/segmento.
6. Si encuentra facturas, valida lote en registro, memo y lineas de factura.
7. Aplica reglas de negocio para decidir liberacion o retencion.

### 5.3 Consultas principales

Consulta de facturas:

- Busca en `CUSTOMRECORD_BIT_REPORTE_CARTERA`.
- Cruza con `transaction` y `transactionLine`.
- Filtra por lote, memo o segmento `cseg2`.
- Limita a centros de costo RETIE o RETILAP.

Consulta de proformas:

- Busca en `CUSTOMRECORD_BIT_REPORTE_PROFORMA`.
- Cruza con transacciones y lineas.
- Filtra por lote, segmento o transaccion origen.
- Limita a centros de costo RETIE o RETILAP.

Consulta adicional de lote en linea:

- Si el lote no aparece en el registro ni en la nota/memo de factura, consulta `transactionLine.cseg2` por id de transaccion y lote.

Las solicitudes HTTP reintentan hasta 3 veces cuando la respuesta es `401`.

## 6. Reglas de negocio identificadas

### 6.1 Criterios que permiten liberar

El robot permite liberar cuando encuentra condiciones como:

- Cliente en lista VIP.
- Centro de costo que permite liberar, especialmente cuando no corresponde a ciertos centros restringidos.
- Factura/proforma sin saldo pendiente y sin condiciones de anulacion que obliguen a retener.
- Lote validado correctamente en registro, memo o linea.

Clientes VIP configurados en el flujo:

- `3175 MASSY ENERGY COLOMBIA SAS`
- `243 APIROS S.A.S.`
- `800 FALABELLA DE COLOMBIA SA`
- `182 AMARILO SAS`
- `707 CONSTRUCTORA COLPATRIA SAS`
- `1003 ERCO ENERGIA S.A.S`

### 6.2 Criterios que retienen

El robot marca retencion cuando detecta, entre otros casos:

- No hay resultados de proforma para el lote.
- La proforma tiene saldo pendiente.
- No hay lote asociado en lineas de factura.
- El lote encontrado en linea no coincide con el lote evaluado.
- La factura tiene saldo y esta vencida.
- La factura tiene saldo y el centro de costo no permite liberar.
- La factura o proforma esta anulada mediante nota de credito.
- La informacion encontrada no permite validar correctamente el lote/factura/proforma.

### 6.3 Resultado escrito en memoria

Si debe retener:

- Columna Q: `RETENIDO`
- Columna R:
  - `RETENIDO POR PAGO/ ...` si `RECHAZO_POR_PAGO = True`
  - `RETENIDO POR FACTURACION/ ...` si `RECHAZO_POR_PAGO = False`
- Columna AD: `SI`
- Columna AE: `No`

Si puede liberar:

- Columna Q: `LIBERADO`
- Columna R: `Liberado por: ...`
- Columna AD: `No`
- Columna AE: `SI`

## 7. Salidas del proceso

### 7.1 Archivo de resultados

El robot exporta el `DataTable` procesado a un archivo Excel con nombre dinamico:

`ResultsyyyyMMddHHmmss.xlsx`

Ruta destino:

`\\10.10.15.120\Automatizaciones\Uipath\liberacion\_dictamenes`

El archivo se escribe en la hoja `Hoja1`.

### 7.2 Actualizacion del Excel maestro

Luego de validar, el robot abre el Excel original en SharePoint:

`Proceso Dictamenes Digital 2024 Segundo Semestre.xlsx`

Por cada fila procesada actualiza:

- Columna Q: estado financiero.
- Columna R: observacion financiera, solo cuando el estado financiero queda `RETENIDO`.
- Columna V: fecha/hora de notificacion, solo despues de un envio exitoso.
- Columna AD: bandera de notificacion para retenidos.
- Columna AE: bandera de notificacion para liberados.

La fila destino se calcula con el indice de fila guardado durante la preparacion del archivo.

### 7.3 Correos electronicos

El robot envia los correos con la actividad Microsoft Office 365 `SendMailConnections`, tomando como referencia el patron usado en el proyecto `robot_transpaso-proformas`.

Configuracion aplicada:

- Buzon remitente: `liberacion.dictamenes@servimeters.com`
- Modo: `UseSharedMailbox=True`
- Conexion OAuth base: `b746ad45-01bd-4f05-bfe8-6284c132303c`
- Actividad anterior reemplazada: `SendMailX` dentro de `Use Outlook 365`.

Si se cambia de ambiente o Studio solicita reasignar la conexion, conservar estos criterios:

- Usar una conexion Microsoft Office 365 con permisos sobre el buzon compartido.
- Mantener el buzon compartido como `Mailbox`.
- Mantener habilitado `UseSharedMailbox`.
- Validar permiso `Send As` o permiso equivalente para el usuario de la conexion.

Asunto:

`Notificacion emision de dictamenes del proyecto {PROYECTO}`

Destinatario:

- Valor tomado de la columna usada como `CurrentRow(19)`.

Plantillas:

- `.data/HtmlContent5.html`: correo para liberados.
- `.data/HtmlContent4.html`: correo para retenidos.

Condiciones de envio:

- Si el estado financiero final es `LIBERADO` y `FECHA NOTIFICACION` esta vacia, crea cuerpo de liberacion.
- Si el estado financiero final es `RETENIDO` y `FECHA NOTIFICACION` esta vacia, crea cuerpo de retencion.
- Si `FECHA NOTIFICACION` ya tiene valor, omite el envio para evitar correos repetidos.

Despues de enviar correctamente:

- Escribe fecha/hora en columna V.
- Deja AD y AE en `No`.
- Escribe temporalmente el cuerpo generado en `body.html`.
- Envia el correo con el cuerpo HTML.

## 8. Requerimientos tecnicos

### 8.1 Plataforma

- UiPath Studio compatible con proyectos Windows.
- Version de Studio observada: `23.12.0.0`.
- Framework objetivo: `Windows`.
- Tipo de ejecucion: `Workflow`.
- Proceso no atendido:
  - `isAttended = false`
  - `requiresUserInteraction = false`
- El proceso es pausble:
  - `isPausable = true`

### 8.2 Paquetes UiPath

Dependencias declaradas en `project.json`:

| Paquete | Version |
| --- | --- |
| UiPath.ConnectionClient | 2.0.12 |
| UiPath.Excel.Activities | 3.2.1 |
| UiPath.IntegrationService.Activities | 1.16.0 |
| UiPath.Mail.Activities | 2.0.11 |
| UiPath.MicrosoftOffice365.Activities | 3.0.14 |
| UiPath.System.Activities | 25.4.2 |
| UiPath.UIAutomation.Activities | 24.10.12 |
| UiPath.WebAPI.Activities | 1.21.1 |

### 8.3 Conexiones y accesos

Se requieren los siguientes accesos:

- Acceso Microsoft 365/SharePoint al archivo de control de dictamenes.
- Conexion UiPath Integration Service para descargar archivos de OneDrive/SharePoint.
- Conexion Outlook 365 para enviar correos.
- Acceso al endpoint SuiteTalk REST de NetSuite.
- Permiso de red y escritura a la ruta compartida de resultados.
- Permisos de lectura/escritura sobre el Excel maestro en SharePoint.

### 8.4 Credenciales

El flujo genera cabecera OAuth 1.0a HMAC-SHA256 para NetSuite. En el XAML se observan valores de consumidor, token, secreto y realm embebidos directamente en actividades `Invoke Code`.

Requerimiento recomendado:

- Migrar esas credenciales a Assets/Credential Assets de UiPath Orchestrator o a una solucion segura equivalente.
- Evitar escribir cabeceras de autorizacion completas en logs.
- Rotar los secretos ya embebidos si este repositorio ha sido compartido fuera del equipo autorizado.

## 9. Requerimientos de datos

Para que el robot opere correctamente:

- La hoja `DOC. SEG. ANTIGUO` debe existir.
- La estructura de columnas debe mantenerse, especialmente columnas A, B, C, D, E, F, G, J, K, Q, R, T, U, AD y AE.
- Los registros a procesar deben tener fecha de emision igual al dia de ejecucion.
- El lote/cotizacion debe estar diligenciado.
- El estado tecnico debe contener `LIBERADO`.
- Si una fila contiene multiples lotes, estos deben venir separados por guion (`-`).
- Los correos de destinatario deben estar diligenciados en la columna que consume el robot.
- NetSuite debe retornar JSON con `totalResults` e `items`.

## 10. Consideraciones operativas

- El proceso mata procesos `EXCEL` al terminar ciertas etapas. Esto puede afectar otros libros abiertos en la misma maquina/robot.
- El formato de fecha usado se construye manualmente y depende del formato esperado en Excel.
- Algunas solicitudes HTTP tienen SSL deshabilitado (`EnableSSLVerification=False`) en ciertas llamadas; conviene revisarlo para ambientes productivos.
- El archivo `orchestrator/assets/assets.json` esta vacio, aunque el proceso depende de credenciales y rutas. Esto sugiere parametrizacion incompleta.
- Existen rutas y enlaces de SharePoint hardcodeados en el XAML.
- La documentacion no replica secretos ni tokens por seguridad.

## 11. Archivos relevantes del proyecto

| Archivo | Descripcion |
| --- | --- |
| `Main.xaml` | Flujo principal del robot. |
| `project.json` | Metadata del proyecto, dependencias y configuracion de runtime. |
| `entry-points.json` | Punto de entrada del proceso. |
| `Proceso Dictamenes Digital.xlsx` | Plantilla/archivo local de referencia para estructura de datos. |
| `body.html` | Archivo temporal o ejemplo de cuerpo HTML generado para correo. |
| `resultados/Results*.xlsx` | Ejemplos/salidas historicas generadas por el robot. |
| `orchestrator/assets/assets.json` | Archivo de assets, actualmente sin definiciones. |

## 12. Riesgos y mejoras recomendadas

- Parametrizar rutas, URLs, cuentas de correo, listas VIP y endpoint de NetSuite.
- Mover secretos de NetSuite a Orchestrator y eliminar credenciales embebidas del XAML.
- Evitar loguear autorizaciones OAuth completas.
- Reemplazar indices de columna por nombres de columna donde sea posible.
- Revisar la correspondencia de columnas T/U para destinatario y enlace, porque la plantilla local muestra nombres que podrian no coincidir con el uso actual del flujo.
- Controlar mejor el cierre de Excel para no terminar sesiones ajenas.
- Validar que las plantillas HTML `.data/HtmlContent4.html` y `.data/HtmlContent5.html` esten incluidas al publicar el paquete.
- Agregar manejo explicito de errores cuando NetSuite retorna JSON invalido, `401` persistente o respuestas sin `items`.

## 13. Bitacora de mejoras y cambios

Este espacio queda reservado para documentar las mejoras evolutivas del robot. Cada ajuste debe registrarse con fecha, caso, causa, decision tecnica, archivos modificados y validacion realizada.

| Fecha | Caso | Estado | Descripcion | Decision / siguiente accion |
| --- | --- | --- | --- | --- |
| 2026-08-24 | Correos duplicados de liberacion | Aplicado | Se evidencia que una misma cotizacion puede recibir correo de liberacion en la manana y nuevamente en la tarde. Ejemplo reportado: cotizacion `201868`, proyecto `SFV DANIEL LEMAITRE`, con correos enviados el 11/08/2026 a las 11:04 AM y 4:05 PM. | Se reordeno el flujo para enviar correos antes de copiar al Excel maestro, se evita enviar cuando no hay bandera pendiente y, despues de un envio exitoso, se limpian las banderas AD/AE en memoria para que el Excel quede en `No`. |
| 2026-08-24 | Ruta de reporte de resultados | Aplicado | El archivo `ResultsyyyyMMddHHmmss.xlsx` debia generarse en una nueva carpeta compartida. | Se cambio `RUTA_RESULTS_SERVER` a `\\10.10.15.120\Automatizaciones\Uipath\liberacion\_dictamenes`. |
| 2026-08-24 | Buzon de envio de correos | Aplicado | Se requiere que las notificaciones salgan desde el buzon `liberacion.dictamenes@servimeters.com` en lugar de la conexion anterior de tesoreria. | Se reemplazo el envio `SendMailX` del bloque `Use Outlook 365` por `SendMailConnections`, usando el patron del proyecto `robot_transpaso-proformas`: `Mailbox=liberacion.dictamenes@servimeters.com`, `UseSharedMailbox=True` y conexion OAuth base `b746ad45-01bd-4f05-bfe8-6284c132303c`. Si falla por permisos, validar `Send As` del usuario asociado a esa conexion sobre el buzon compartido. |
| 2026-08-24 | Condicion de envio antes de actualizar estado | Aplicado | En prueba, el Excel maestro podia quedar como `LIBERADO` sin enviar correo porque la bandera `AE` ya venia en `No` y no se reactivaba. | Se cambio la decision de envio para depender del estado financiero final y de `FECHA NOTIFICACION` vacia. Si envia correctamente, escribe fecha/hora en la columna V y luego copia el estado al Excel maestro. |
| 2026-08-25 | Descarte temprano de dictamenes ya notificados | Aplicado | En prueba se observo que registros con `FECHA NOTIFICACION` diligenciada llegaban hasta `Validar lotes facturados` y consultaban NetSuite aunque ya no debian enviar correo. | Se agrego un filtro inicial sobre la columna V / `CurrentRow(21)` para conservar solo filas con `FECHA NOTIFICACION` vacia antes de validar lotes facturados. |

## 14. Analisis de mejora - evitar correos repetidos

### 14.1 Comportamiento actual observado

Antes del ajuste, el robot decidia el envio usando estas banderas del `DataTable`:

- Columna AD / `CurrentRow(29)`: `NOTIFICACION RETENIDOS`.
- Columna AE / `CurrentRow(30)`: `NOTIFICACION LIBERADOS`.

Durante la validacion financiera:

- Si el dictamen queda retenido, asigna `AD = SI` y `AE = No`.
- Si el dictamen queda liberado, asigna `AD = No` y `AE = SI`.

Despues, en la etapa `Copiar datos al Excel`, esas banderas se escriben en el Excel maestro. Finalmente, en la etapa `Enviar correos`, el robot envia:

- Correo de liberacion si `AE = SI`.
- Correo de retencion si `AE = NO` y `AD = SI`.

El problema era que el envio ocurria despues de haber escrito `AE = SI` o `AD = SI` en el Excel maestro. Si el robot volvia a ejecutarse el mismo dia, el registro podia quedar nuevamente con bandera de notificacion activa y enviar otro correo.

### 14.2 Causa probable

No existe una marca persistente de correo enviado. Las columnas AD y AE funcionan como banderas para disparar el correo, pero no se limpian ni cambian a un estado como `ENVIADO` despues de un envio exitoso.

### 14.3 Cambio aplicado

Usar `FECHA NOTIFICACION` como marca persistente de correo enviado:

- Vacia: pendiente de notificacion si el estado financiero final es `LIBERADO` o `RETENIDO`.
- Con fecha/hora: correo ya enviado, no repetir.

Despues del envio exitoso:

- Registrar fecha/hora en columna V.
- Actualizar `AE` y `AD` a `No` en el Excel maestro.

### 14.4 Opcion tecnica aplicada

Se reordeno el flujo para que el envio de correos ocurra antes de copiar las banderas definitivas al Excel maestro:

1. Validar registros y calcular estados.
2. Enviar correos segun estado financiero final y `FECHA NOTIFICACION` vacia.
3. Si el envio fue exitoso, escribir fecha/hora en memoria y limpiar `AD`/`AE`.
4. Copiar al Excel maestro el estado financiero, observacion, fecha de notificacion y banderas ya limpias.

Esto evita abrir el Excel maestro dos veces y deja el archivo en un estado consistente para la siguiente ejecucion.

Adicionalmente, el envio ya no depende de que `AD` o `AE` vengan vacias, porque el documento puede traerlas por defecto en `No`. El control persistente de no repeticion queda en `FECHA NOTIFICACION`: si esta vacia, se envia; si ya tiene fecha/hora, se omite el correo.

### 14.5 Opcion tecnica alternativa

Mantener el orden actual y agregar una actualizacion posterior al envio:

1. Enviar correo.
2. Abrir el Excel maestro nuevamente.
3. Escribir `No` en AD o AE de la fila correspondiente.

Desventaja: aumenta tiempo de ejecucion y riesgo de bloqueo del archivo Excel/SharePoint.

### 14.6 Validaciones sugeridas

- Ejecutar el robot dos veces el mismo dia con la misma cotizacion liberada.
- Confirmar que en la primera ejecucion se envia un solo correo.
- Confirmar que en la segunda ejecucion no se vuelve a enviar correo si la columna V ya tiene fecha/hora.
- Confirmar que los retenidos mantienen el mismo comportamiento.
- Revisar que los resultados financieros Q/R sigan actualizandose correctamente.
