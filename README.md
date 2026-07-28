# Ecosistema de IA para Facturas de Proveedores

## 1. Caso de uso

Este proyecto automatiza la recepción, extracción, validación y aprobación de facturas de proveedores.

El flujo recibe una factura por Gmail, valida que exista un archivo adjunto compatible, envía el documento a OpenAI para extraer información estructurada, aplica reglas de validación, registra la trazabilidad en Airtable y solicita una aprobación humana antes de completar el proceso.

También incluye rutas de excepción para:

- Facturas con datos incompletos o inconsistentes.
- Rechazo por parte del aprobador.
- Errores técnicos en la llamada a OpenAI.
- Registro de incidencias y notificaciones automáticas.

## 2. Arquitectura

La solución integra los siguientes componentes:

- **Gmail:** canal de entrada de facturas y envío de notificaciones.
- **n8n:** orquestador del flujo, validaciones, decisiones y manejo de errores.
- **OpenAI:** extracción de datos desde archivos PDF, JPG o PNG.
- **Airtable:** base de datos para facturas, proveedores, órdenes de compra, aprobaciones y logs de errores.
- **Formulario de n8n / nodo Wait:** aprobación o rechazo humano.

### Flujo general

1. Gmail detecta un correo etiquetado como `Facturas`.
2. n8n obtiene el mensaje y descarga el archivo adjunto.
3. Se valida la existencia y el formato del archivo.
4. El archivo se carga y se procesa con OpenAI.
5. La respuesta de IA se interpreta y normaliza.
6. Se validan campos obligatorios, importes, fechas, moneda, categoría y confianza.
7. La factura se registra en Airtable.
8. Si requiere revisión, se registra el error y se envía una notificación.
9. Si es válida, se crea una aprobación pendiente.
10. El flujo queda pausado hasta recibir la decisión humana.
11. Se actualizan la factura y la aprobación como aprobadas o rechazadas.
12. Se envía la notificación final.
13. Si OpenAI falla, se registra un log técnico y se notifica el error.

## 3. Estructura de Airtable

La base contiene las siguientes tablas:

- **Facturas**
- **Proveedores**
- **Órdenes de compra**
- **Aprobaciones**
- **Logs de errores**

### Vistas públicas de auditoría (solo lectura)

Las siguientes vistas permiten revisar la trazabilidad del proyecto sin permisos de edición:

- [Facturas](https://airtable.com/appAUOggvIu7XpHSt/shraH0bJRKSZKtLDH)
- [Proveedores](https://airtable.com/appAUOggvIu7XpHSt/shruuqbwfGmGFzf1m)
- [Órdenes de compra](https://airtable.com/appAUOggvIu7XpHSt/shrIoex07Letg0hql)
- [Aprobaciones](https://airtable.com/appAUOggvIu7XpHSt/shrpwtjwCPQyfnbwW)
- [Logs de errores](https://airtable.com/appAUOggvIu7XpHSt/shrkTRcEPNRTrVSb9)

Estas vistas fueron configuradas para consulta y auditoría. No otorgan permisos de edición sobre la base original.

## 4. Requisitos para replicar el proyecto

Antes de importar el flujo se necesita:

- Una instancia de n8n.
- Una cuenta de Gmail conectada a n8n.
- Una API Key de OpenAI con saldo disponible.
- Una base de Airtable con las tablas y campos requeridos.
- Credenciales de Airtable configuradas en n8n.
- Una etiqueta de Gmail llamada `Facturas`.

## 5. Pasos de réplica

### 5.1 Importar el flujo

1. Abrir n8n.
2. Crear un flujo nuevo.
3. Seleccionar **Import from File**.
4. Importar el archivo JSON ubicado en `02-flujo-n8n/`.
5. Guardar el flujo.

### 5.2 Configurar credenciales

Asignar las credenciales correspondientes en estos nodos:

- Gmail Trigger.
- Obtener correo y descargar factura.
- Subir un archivo.
- Extraer datos de factura con OpenAI.
- Todos los nodos de Airtable.
- Todos los nodos de Gmail utilizados para notificaciones.

Las credenciales no se incluyen en el repositorio por seguridad.

### 5.3 Configurar Gmail

1. Crear en Gmail una etiqueta llamada `Facturas`.
2. En el nodo **Recepción de factura por Gmail**, seleccionar esa etiqueta.
3. Verificar que el evento sea `Message Received`.
4. Definir la frecuencia de consulta.

### 5.4 Configurar OpenAI

En el nodo **Extraer datos de factura con OpenAI**:

- Endpoint: `https://api.openai.com/v1/responses`
- Modelo: `gpt-4.1-mini`
- Límite: `max_output_tokens = 1000`
- Credencial: cuenta de OpenAI configurada en n8n.

### 5.5 Configurar Airtable

En cada nodo de Airtable se deben reemplazar los valores de configuración por los correspondientes a la base utilizada.

#### Valores que deben configurarse

| Placeholder o referencia | Debe reemplazarse por |
|---|---|
| `AIRTABLE_BASE_ID` | ID real de la base de Airtable |
| `FACTURAS_TABLE_ID` | ID o nombre de la tabla Facturas |
| `APROBACIONES_TABLE_ID` | ID o nombre de la tabla Aprobaciones |
| `LOGS_TABLE_ID` | ID o nombre de la tabla Logs de errores |
| `PROVEEDORES_TABLE_ID` | ID o nombre de la tabla Proveedores |
| `ORDENES_COMPRA_TABLE_ID` | ID o nombre de la tabla Órdenes de compra |
| `AIRTABLE_CREDENTIAL` | Credencial de Airtable creada en n8n |

No deben quedar valores genéricos como `BASE_ID`, `TABLE_ID`, `RECORD_ID`, `PLACEHOLDER`, `TODO` o campos vacíos sin explicación.

### 5.6 Mapeo de los nodos de Airtable

#### Guardar factura

Debe crear un registro en la tabla **Facturas** y mapear, como mínimo:

- Número de factura.
- Proveedor.
- CUIT.
- Fecha de emisión.
- Fecha de vencimiento.
- Moneda.
- Subtotal.
- IVA.
- Otros impuestos.
- Total.
- Orden de compra.
- Categoría.
- Resumen de IA.
- Confianza de extracción.
- Estado.
- Motivo de revisión.
- Thread ID o ID del hilo de Gmail.

#### Crear aprobación pendiente

Debe crear un registro en **Aprobaciones** con:

- Factura vinculada.
- Estado `Pendiente`.
- Fecha de solicitud.
- Identificador de la ejecución.
- Correo o nombre del aprobador, cuando corresponda.

#### Actualizar factura pendiente de aprobación

Debe utilizar el ID del registro creado en **Guardar factura** y actualizar:

- Estado `Pendiente de aprobación`.

#### Actualizar aprobación aprobada o rechazada

Debe utilizar el ID del registro creado en **Crear aprobación pendiente** y actualizar:

- Estado.
- Fecha de respuesta.
- Comentario del aprobador.
- Decisión.

#### Actualizar factura aprobada o rechazada

Debe utilizar el ID de la factura y actualizar:

- Estado final.
- Fecha de aprobación o rechazo.
- Comentario, cuando corresponda.

#### Registrar error de validación

Debe crear un registro en **Logs de errores** con:

- Factura vinculada.
- Tipo de error.
- Etapa.
- Descripción.
- Fecha y hora.
- Estado del error.

#### Registrar error técnico de OpenAI

Debe crear un registro en **Logs de errores** con:

- Tipo `Error técnico IA`.
- Nodo o etapa.
- Mensaje del error.
- Fecha y hora.
- Estado `Pendiente`.

## 6. Trazabilidad entre registros

Los nodos de actualización no deben utilizar IDs escritos manualmente.

Los IDs deben provenir dinámicamente de los nodos anteriores, por ejemplo:

- ID de factura: salida del nodo **Guardar factura**.
- ID de aprobación: salida del nodo **Crear aprobación pendiente**.
- ID del hilo: salida del nodo de Gmail.
- Datos validados: salida del nodo **Validar datos extraídos de la factura**.

Ejemplo conceptual de expresión en n8n:

```text
{{ $('Guardar factura').item.json.id }}
```

El nombre exacto del campo puede variar según la respuesta del nodo de Airtable, por lo que debe verificarse en la salida de ejecución.

## 7. Datos de prueba

El repositorio utiliza datos ficticios.

Para probar el flujo:

1. Enviar un correo a la cuenta conectada.
2. Agregar la etiqueta `Facturas`.
3. Adjuntar una factura PDF, JPG o PNG.
4. Verificar la ejecución en n8n.
5. Revisar el registro en Airtable.
6. Completar el formulario de aprobación.
7. Confirmar la actualización final y la notificación.

## 8. Pruebas contempladas

- Factura válida y aprobada.
- Factura válida y rechazada.
- Factura con total inconsistente.
- Error técnico de OpenAI.
- Archivo ausente o formato no permitido.

## 9. Seguridad

- Las credenciales y API Keys no se incluyen en el repositorio.
- Los IDs privados pueden estar reemplazados por placeholders claramente documentados.
- Los datos de las facturas utilizadas son ficticios.
- No se deben publicar URLs de reanudación del nodo Wait.
- No se deben publicar correos personales, tokens ni webhooks privados.

## 10. Contenido del repositorio

- `01-arquitectura/`: diagrama de arquitectura.
- `02-flujo-n8n/`: exportación JSON del flujo.
- `03-documentacion/`: documento principal de la entrega.
- `04-evidencias/`: capturas del funcionamiento.
- `05-pruebas/`: evidencias de los casos de prueba.
- `06-video/`: video o enlace a la demostración.

## 11. Consideraciones

Este proyecto corresponde a un MVP académico. Para una implementación productiva se recomienda incorporar validación real contra proveedores y órdenes de compra, detección de duplicados, control de permisos, monitoreo, reintentos y conexión con un ERP.
