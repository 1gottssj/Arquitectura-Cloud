# Hito 2 — Plataforma B2B de procesamiento y analítica post-llamada

**Asignatura:** TICS400 — Arquitectura Cloud  
**Integrantes:** Florencia Arriagada, Bastián de la Fuente y Joaquín Urrutia  
**Profesor:** Gonzalo Huerta  
**Versión consolidada:** 3 de noviembre de 2026  
**Proveedor:** AWS 
**Entorno local:** Ministack

Este documento reúne el alcance, las decisiones de arquitectura, la seguridad y los resultados de validación disponibles.

## Índice

1. [Necesidad, usuarios y alcance](#1-necesidad-usuarios-y-alcance)
2. [Resultados del servicio](#2-resultados-del-servicio)
3. [Reglas de análisis y métricas](#3-reglas-de-análisis-y-métricas)
4. [Datos de entrada y salida](#4-datos-de-entrada-y-salida)
5. [ETL y consultas analíticas](#5-etl-y-consultas-analíticas)
6. [Escenarios de carga](#6-escenarios-de-carga)
7. [Arquitectura y resiliencia](#7-arquitectura-y-resiliencia)
8. [Seguridad y políticas IAM](#8-seguridad-y-políticas-iam)
9. [Validación realizada en Ministack](#9-validación-realizada-en-ministack)
10. [Soportes y pendientes](#10-soportes-y-pendientes)

# 1. Necesidad, usuarios y alcance

## 1.1. Necesidad de negocio y propuesta de valor

El caso de estudio considera empresas que ya graban sus llamadas, pero necesitan convertir ese audio en información organizada y comparable. Obtener transcripciones, identificar motivos de contacto y consolidar resultados requiere almacenamiento, procesamiento y operación tecnológica.

La propuesta es una plataforma B2B que permite **externalizar ese proceso completo**, en lugar de contratar solamente la conversión de audio a texto. Cada empresa recibe transcripciones y métricas para sus propios análisis. No se dispone de mediciones de clientes reales que permitan afirmar un ahorro o un impacto cuantificado.

## 1.2. Usuarios

| Actor | Uso previsto |
|---|---|
| Empresa cliente y su sistema de carga | Entregar grabaciones finalizadas y configurar los temas de interés. |
| Supervisor, analista de datos o de calidad | Consultar transcripciones e indicadores de su empresa. |
| Administración técnica | Supervisar trabajos, atender errores y mantener la plataforma. |

Cada empresa accederá únicamente a sus propios datos. La administración técnica no implica acceso irrestricto al contenido de las conversaciones.

## 1.3. Alcance acordado

Se recibirán grabaciones finalizadas, se procesarán de forma asíncrona y se generarán transcripciones, una categoría principal de interacción, un tema principal y términos relevantes. Los resultados se consolidarán por empresa y período.

**La transcripción utilizará Whisper/faster-whisper. La clasificación del texto utilizará código con reglas y configuración por empresa, no un LLM ni otro modelo de IA para clasificar.** El programa será común; cambiarán los catálogos y parámetros de cada cliente. La elección del método está tomada, aunque faltan detalles de reglas y evaluación.

Quedan fuera telefonía, grabación de llamadas, streaming en tiempo real, CRM, chatbots, sentimiento, evaluación automática del desempeño de agentes y recomendaciones de gestión. Tampoco se comprometen un dashboard completo, un portal de configuración ni el descubrimiento automático de temas nuevos.

# 2. Resultados del servicio

| Resultado | Qué recibe el cliente |
|---|---|
| Transcripción individual | Texto asociado a su empresa y llamada, con metadatos y acceso autorizado. |
| Tipo de interacción | Una categoría: consulta, reclamo, solicitud, mixta o indeterminada. |
| Tema principal | Un tema de su catálogo o «Tema no identificado». |
| Términos relevantes | Apariciones de palabras o expresiones y número de llamadas que las contienen. |
| Analítica agregada | Distribuciones, cruces entre tipo y tema y comparaciones temporales. |
| Información operacional | Volumen procesado, duración, tiempos de ejecución y trabajos fallidos. |

El tipo describe la naturaleza de la interacción; el tema, el asunto tratado. Por ejemplo, «me cobraron dos veces» puede corresponder a **reclamo** y **facturación**, si ese tema existe en el catálogo. Es un ejemplo funcional, no una clasificación ejecutada.

El cliente interpreta los resultados. Ni una etiqueta de reclamo ni la duración de una grabación demuestran por sí solas la calidad de atención o la resolución del problema.

# 3. Reglas de análisis y métricas

## 3.1. Clasificación de la interacción

| Categoría | Criterio funcional |
|---|---|
| Consulta | Obtener información, orientación o aclaraciones, sin una gestión o disconformidad como propósito principal. |
| Reclamo | Comunicar un problema, incumplimiento o disconformidad. Pedir que se resuelva puede formar parte del mismo reclamo. |
| Solicitud | Pedir una acción, trámite o modificación sin que un problema sea el motivo principal. |
| Mixta | Existen motivos independientes de distintos tipos y no se reconoce uno predominante. |
| Indeterminada | El contenido disponible no permite establecer el propósito con suficiente claridad. |

La clasificación considerará el contexto y la relación entre intenciones. Pedir información sobre un trámite es una consulta; pedir que la empresa lo ejecute es una solicitud, salvo que forme parte de otro motivo principal.

Una llamada recibe **una sola categoría final**. «Mixta» no se usa por cualquier combinación de preguntas y peticiones, sino por motivos independientes sin uno principal. Un fallo técnico no se convierte en una clasificación indeterminada.

## 3.2. Tema principal y términos

Cada empresa tendrá un catálogo versionado con identificador, nombre, descripción y ejemplos de cada tema. Para el prototipo se propone un catálogo reducido de cuatro a seis temas, entregado en un archivo estructurado como CSV y revisado antes de utilizarlo.

El análisis aplicará la configuración de la empresa de la llamada y asignará **un solo tema principal**. Si ninguno encaja o no hay predominio claro, devolverá «Tema no identificado». Los catálogos no se redefinen por llamada ni se comparten entre empresas sin autorización.

Las reglas concretas de puntuación, desempate, normalización y exclusión de información personal deben completarse en la configuración del programa. La presencia de una palabra no demuestra por sí sola el tema principal. Contar términos tampoco equivale a identificar automáticamente preguntas frecuentes completas.

## 3.3. Cálculo de indicadores

Para una empresa y período, `N` representa las llamadas únicas con resultados completos y vigentes. Se incluyen las categorías inciertas; se excluyen pendientes y fallos técnicos de las métricas de contenido.

| Indicador | Regla |
|---|---|
| Porcentaje por tipo o tema | Llamadas de esa categoría o tema / N × 100. |
| Cruce tipo–tema | Número de llamadas que cumplen ambas condiciones. |
| Reclamos de un tema dentro de los reclamos | Reclamos del tema / total de reclamos × 100. |
| Menciones de un término | Suma de sus apariciones. |
| Llamadas con un término | Número de llamadas distintas que lo contienen. |
| Horas de audio | Suma de duraciones en segundos / 3.600. |
| Duración promedio | Suma de duraciones / número de grabaciones incluidas. |
| Tiempo de procesamiento | Tiempo entre inicio efectivo y finalización; espera en cola diferenciada. |
| Tasa de trabajos fallidos | Trabajos únicos finalizados en error / trabajos únicos finalizados × 100. |

Los porcentajes indican siempre su denominador. Sin observaciones no se calcula un porcentaje o promedio. En un conjunto no vacío, las cinco categorías de interacción suman el total, al igual que los temas más «Tema no identificado».

Las métricas del contenido utilizan la fecha original de la llamada y la zona horaria de la empresa; las operacionales, las fechas de ejecución. Reintentar no crea otra llamada. Reprocesar puede crear un trabajo y una versión nuevos, pero no otra conversación.

Para comparar períodos se usan diferencia absoluta, variación relativa cuando el valor anterior no sea cero y puntos porcentuales para diferencias entre proporciones. Se deben considerar cambios de volumen, catálogos y reglas, sin atribuir automáticamente causas.

## 3.4. Evaluación prevista

El tipo de interacción y el tema se evaluarán por separado mediante conversaciones autorizadas o ficticias con etiquetas revisadas por personas. Los ejemplos usados para ajustar las reglas no serán los únicos usados para evaluarlas.

Se revisarán exactitud, precisión, exhaustividad y F1 por categoría, matriz de confusión y proporción de resultados inciertos. Comparar textos revisados con transcripciones automáticas permitirá examinar también el efecto de errores de reconocimiento. Se conservarán las versiones del método y del catálogo.


# 4. Datos de entrada y salida

| Información | Definición |
|---|---|
| Audio y fecha de llamada | La empresa aporta una grabación finalizada y su fecha/hora. |
| `company_id` | El backend lo determina a partir de una identidad autorizada, no de un valor arbitrario enviado por el cliente. |
| `call_id` | Identificador interno generado por la plataforma. Un identificador externo puede conservarse como `external_call_id` |
| Catálogo de temas | Configuración versionada por empresa, independiente de cada carga. |
| Transcripción | Texto completo almacenado en S3; DynamoDB conserva su referencia. |
| Resultado estructurado | Tipo, tema, términos y frecuencias, duración, estado y versiones. |
| Registro de ejecución | Información para seguir trabajos, errores y tiempos. |

El audio original, segmentos, transcripciones y catálogos se conservan en S3 según su finalidad. DynamoDB mantiene estado y resultados compactos. Los datos analíticos preparados se guardan en S3 como Parquet.

La combinación de empresa y llamada identifica la conversación. La versión del trabajo y el índice de segmento distinguen las unidades de ejecución. Una carga repetida con la misma clave de idempotencia debe recuperar la llamada existente; un contenido diferente con esa identidad debe tratarse como conflicto. Generar un identificador nuevo no resuelve por sí solo la deduplicación.

SQS transporta **referencias al trabajo y a los objetos**, no el audio completo. Se conserva la relación entre entrada, transcripción, resultado vigente y configuración aplicada.

# 5. ETL y consultas analíticas

**Glue Job ejecutará un ETL diario; Glue Data Catalog describirá las tablas; Athena realizará las consultas.** No son tres nombres para la misma función. El horario UTC concreto y los recursos del job quedan por configurar. [A9–A12]

El ETL extrae resultados estructurados, selecciona versiones completas y vigentes, evita duplicados y organiza por empresa y fecha. Publica un nuevo conjunto Parquet —snapshot— sin volver a transcribir ni clasificar audios.

La publicación se actualiza solo cuando el conjunto esté completo. Si falla, se conserva el anterior. Las consultas de llamadas y términos deben utilizar una misma versión publicada, sin sumar snapshots antiguos. El reporte indica su fecha de actualización; la extracción paginada no se presenta como una instantánea transaccional de una base que sigue cambiando.

| Consulta | Utilidad |
|---|---|
| Distribución por interacción | Cantidad y porcentaje de consultas, reclamos, solicitudes, mixtas e indeterminadas. |
| Distribución por tema | Cantidad y porcentaje por tema, incluidos los no identificados. |
| Cruce de tipo y tema | Identificar, por ejemplo, los temas principales de los reclamos. |
| Frecuencia de términos | Distinguir menciones totales de llamadas que contienen una expresión. |
| Evolución temporal | Comparar los indicadores entre períodos. |

También se podrán consultar las métricas operacionales. Los resultados se entregarán mediante consultas predefinidas por una API autorizada, sin acceso directo del cliente a credenciales AWS o SQL arbitrario. No se compromete QuickSight. Los audios y textos completos permanecen separados de los datos agregados.

# 6. Escenarios de carga

Como referencia de escala, la investigación aportada utiliza el informe RTA 2023–24: **1.242 llamadas atendidas por día laborable y 9 minutos y 30 segundos de conversación promedio**. Ese caso no representa a nuestros futuros clientes ni mide la duración de sus archivos. [D1]

El escenario elegido considera **dos empresas, 500 grabaciones diarias por empresa, diez minutos de duración media y 22 días de actividad mensual**. Son supuestos de diseño, no mediciones. Los diez minutos son una aproximación de referencia, no un máximo por llamada.

| Magnitud | Normal | Pico |
|---|---:|---:|
| Grabaciones diarias conjuntas | 1.000 | 1.000 |
| Ventana de entrega | 8 horas | 1 hora |
| Tasa media de ingreso | 125 archivos/hora | 1.000 archivos/hora |
| Grabaciones mensuales | 22.000 | 22.000 |

**El pico concentra la misma carga, no triplica el volumen mensual ni implica llamadas simultáneas.** Recibir archivos durante una hora tampoco exige transcribirlos dentro de esa misma hora.

El volumen mensual equivale a 3.666,7 horas de audio. Como referencia de almacenamiento se utiliza WAV PCM sin compresión, 8 kHz, 16 bits y dos canales: `600 × 8.000 × 16 × 2 / 8 = 19.200.000 bytes` por diez minutos, aproximadamente 19,2 MB y 422,4 GB nuevos al mes. Son unidades decimales; no incluyen cabeceras, transcripciones, respaldos ni otros resultados. No se incorpora Amazon Connect al diseño por utilizar su referencia de formato. [D2–D4]

Siguen como propuestas operacionales la retención de 30 días de audio y 90 de transcripciones/resultados, y completar un lote pendiente de 1.000 grabaciones en cuatro horas sin nuevas entradas. No son obligaciones normativas ni capacidades medidas. Las horas de audio no son horas de cómputo. El escenario y sus límites se conservan para dimensionamiento, sin presupuesto en esta exposición.

# 7. Arquitectura y resiliencia

## 7.1. Servicios y recorrido

| Capa | Componentes elegidos | Función |
|---|---|---|
| Ingesta y almacenamiento | Cognito, API Gateway, Lambda de ingesta y S3 | Autorizar la carga y conservar objetos. |
| Eventos y colas | Notificaciones S3, SQS y una DLQ por cola | Entregar referencias y recuperar trabajos fallidos. |
| Cómputo | Lambda de segmentación, transcripción y cierre/análisis | Preparar segmentos, transcribir y analizar la conversación completa. |
| Estado | DynamoDB | Mantener identidad, estado, versiones y referencias a resultados. |
| Analítica | Glue Job, S3 Parquet, Glue Data Catalog, Athena y Lambda de consulta | Preparar y servir indicadores autorizados. |

La empresa se autentica y registra la llamada por API. El backend obtiene su identidad autorizada y entrega una URL prefirmada; el audio se carga directamente a S3, no atraviesa API Gateway. La notificación de creación inicia el procesamiento mediante SQS. [A5–A8, A13–A14]

El equipo conserva sus vistas de arquitectura de servicios y de red. Ambas deben ser coherentes con esta selección: una explica responsabilidades y flujos; la otra, conectividad y fronteras. Este texto no sustituye ni declara validado el diagrama de red. [C1–C2]

## 7.2. Cómputo seleccionado: Lambda

La decisión de cómputo está tomada: **AWS Lambda**, con trabajo dividido por eventos. La justificación del equipo es acotar cada ejecución, integrarla con SQS y permitir concurrencia configurable sin administrar los hosts. El diseño segmentado añade coordinación que debe implementarse y probarse. [A1–A2, A15]

| Etapa | Trabajo |
|---|---|
| Segmentación — Q-ingesta | Crear un manifiesto de segmentos esperados, guardar fragmentos y enviar sus referencias a Q-segmentos. |
| Transcripción — Q-segmentos | Procesar un segmento por invocación, conservar el texto parcial y avisar a Q-cierre. |
| Cierre/análisis — Q-cierre | Comprobar que están todos los segmentos, ordenar y recomponer el texto, aplicar las reglas y publicar el resultado. |

El análisis se hace sobre **la conversación completa**, no votando categorías independientes de los segmentos. Deben revisarse límites y solapes de audio para evitar pérdidas o repeticiones en la recomposición. Tamaño del segmento, modelo concreto, memoria, timeout y concurrencia necesitan ensayo; no se ha probado Whisper en esta instalación.

## 7.3. Reintentos y recuperación

| Situación | Respuesta prevista |
|---|---|
| Fallo temporal o timeout | No devolver éxito; el mensaje puede volver a estar disponible para reintento. |
| Segmento ya completado | Reutilizar su resultado y reenviar la notificación de continuación. |
| Fallos reiterados | Derivar a la DLQ, alertar y revisar antes de reenviar. |
| Fallo tras guardar, pero antes de avisar | Reintentar la continuación sin perder el resultado guardado. |
| Falta un segmento | No publicar la llamada como completa. |
| Falla un reprocesamiento o el ETL | Conservar la versión completa publicada anteriormente. |

La configuración inicial propuesta usa `BatchSize = 1`, ventana de agrupación cero y `maxReceiveCount = 5`. Cada cola tendrá una DLQ. La visibilidad se configurará al menos a seis veces el timeout de la función consumidora, según la referencia utilizada. Timeout, retenciones y alarmas se ajustarán con las pruebas.

La función devuelve éxito después de guardar resultados y enviar la continuación correspondiente; la integración SQS–Lambda confirma el mensaje. No se deben ocultar errores devolviendo éxito. S3, DynamoDB y SQS no se consideran una transacción única.

## 7.4. Consistencia de resultados

El trabajo se identifica por empresa, llamada, versión e índice de segmento. Una escritura condicional en DynamoDB asigna un propietario y un vencimiento. Si vence, otro intento puede recuperar el trabajo; un intento antiguo no debe publicar sobre el nuevo. Los resultados de cada intento se guardan separados y solo el propietario vigente publica la referencia válida. [A3]

El cierre cuenta **segmentos únicos completados**, no mensajes recibidos. Recompone y publica únicamente cuando está completo el manifiesto. Un reprocesamiento crea otra versión de la misma llamada; la anterior permanece vigente hasta terminar correctamente la nueva.

Son reglas del diseño, todavía no verificadas mediante fallos o concurrencia. No se promete ejecución exactamente una vez ni que crear una tabla DynamoDB implemente automáticamente este protocolo.

# 8. Seguridad y políticas IAM

## 8.1. Identificación y autorización

| Actor | Acceso diseñado |
|---|---|
| Aplicación del call center | Cognito M2M con Client Credentials; asociación interna de la identidad de aplicación con su empresa. |
| Supervisor o analista | Cognito para usuarios, código de autorización con PKCE; asociación interna del usuario con empresa y rol. |

API Gateway valida el token y los scopes; Lambda consulta la asociación interna y comprueba que el recurso solicitado pertenezca a la empresa autorizada. Las funciones de negocio no pueden modificar esa asociación. No se acepta un `company_id` arbitrario ni se supone que cualquier token contiene automáticamente la empresa correcta. [A5–A7]

Autenticarse identifica al actor; **autorizar determina qué puede consultar**. Las comprobaciones deben abarcar llamadas, transcripciones y resultados de consultas. El cliente no recibe credenciales AWS ni SQL libre. Separar carpetas no demuestra aislamiento.

Se prevén buckets privados, HTTPS/TLS, protección en reposo y logs sin transcripciones, tokens o secretos. Las URLs prefirmadas se limitan a objetos y vencimientos concretos; no se describen como enlaces de un solo uso. Estos son controles de diseño, no resultados de las pruebas locales. [A8]

## 8.2. Permisos mínimos por función

| Función o servicio | Permisos funcionales | Límite |
|---|---|---|
| Lambda de ingesta | Leer identidad autorizada, registrar llamada y preparar carga S3. | No leer todas las transcripciones. |
| Lambda de segmentación | Leer audio, escribir segmentos y enviar trabajos. | No consultar analítica. |
| Lambda de transcripción | Consumir su cola, leer segmentos, escribir parciales y actualizar estado. | Catálogo y recursos ajenos a su función excluidos. |
| Lambda de cierre/análisis | Leer parciales y catálogo, recomponer y publicar resultados. | Catálogo de solo lectura. |
| Glue Job | Leer resultados estructurados, escribir Parquet y actualizar el catálogo previsto. | No leer audios ni transcripciones completas. |
| Lambda de consulta | Autorizar al solicitante y consultar datos, catálogo y resultados. | Sin administración de datos ni acceso directo AWS para clientes. |

Son roles internos que pueden procesar varias empresas; el backend mantiene la autorización de cada usuario. IAM no reemplaza esa comprobación de propiedad. Los resultados de Athena también requieren protección. [A4, A10–A12]

## 8.3. Plantillas JSON disponibles

El paquete de soporte contiene seis políticas de ejecución, dos políticas de confianza y cuatro plantillas para permisos entre servicios y transporte:

| Archivos de soporte | Propósito |
|---|---|
| `iam/01_lambda_ingesta.json` a `iam/06_lambda_consulta.json` | Roles de ingesta, segmentación, transcripción, cierre, ETL y consulta. |
| `iam/07_trust_lambda.json`, `iam/08_trust_glue.json` | Indicar qué servicio puede asumir los roles correspondientes. |
| `iam/09_sqs_ingesta_resource_policy.json` | Permitir notificaciones S3 hacia SQS, limitadas a origen y cuenta. |
| `iam/10_lambda_ingesta_invoke_policy.json`, `iam/11_lambda_consulta_invoke_policy.json` | Limitar invocaciones desde API Gateway a los recursos previstos. |
| `iam/12_s3_deny_insecure_transport.template.json` | Plantilla de restricción de transporte para adaptar a los buckets. |

Fragmento de `iam/03_lambda_transcripcion.json`:

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::${BUCKET_WORK}/segments/*"
}
```

Autoriza lectura de objetos del prefijo indicado; **no es la política completa ni una prueba de separación entre clientes**. Las políticas completas y sus parámetros acompañan al documento. No basta con tener un JSON válido: hay que asociarlo al rol o recurso adecuado y comprobar acciones y restricciones. [A4, A13–A14]

Las plantillas conservan parámetros de cuenta, región y recursos por completar. No se han aplicado ni verificado en las pruebas documentadas aquí. La sintaxis comprobable del archivo no acredita mínimo privilegio efectivo ni seguridad del despliegue.

# 9. Validación realizada en Ministack

## 9.1. Entorno utilizado

Las comprobaciones siguientes se ejecutaron el **3 de noviembre de 2026 en el computador de la integrante**, y sus comandos y salidas se compartieron durante el trabajo. No se ejecutaron desde el asistente ni son salidas fabricadas.

| Elemento | Valor observado |
|---|---|
| Sistema | Windows y PowerShell. |
| Docker Desktop / Engine | 4.76.0 / 29.5.2; contenedores Linux. |
| AWS CLI | 2.39.3. |
| Ministack | 1.5.36, edición `full`, contenedor `healthy`. |
| Endpoint | `http://127.0.0.1:4566`, acceso local. |
| Credenciales de prueba | Valores ficticios; no credenciales de AWS real. |
| Autorización del simulador | `auth: disabled`. |

## 9.2. Comprobaciones y resultados

| Comprobación ejecutada | Evidencia observada | Alcance del resultado |
|---|---|---|
| Inicio y comunicación | Docker informa `healthy`; el endpoint de salud responde. | Entorno local accesible. Que un servicio figure `available` no prueba sus operaciones. |
| S3 | Bucket creado, archivo subido y listado, descarga y comparación de hashes: `VALIDACION S3 EXITOSA`. | Almacenamiento y recuperación íntegra del archivo de prueba. |
| SQS | Cola creada; envío y recepción con el mismo `MessageId`; eliminación solicitada sin error y recepción posterior vacía. | Ciclo básico de mensajería en el caso probado. |
| Lambda | Función activa; invocación directa con `StatusCode: 200`, sin `FunctionError`, y respuesta `{"ok": true, "call_id": "prueba-001"}`. | Ejecución del código sencillo de prueba. |
| DynamoDB | Tabla activa; escritura manual y recuperación de `call_id=prueba-001` y `estado=COMPLETED`. | Escritura y lectura; no es una transcripción completada. |
| SQS → Lambda | Mapeo `Enabled`, registro de procesamiento de un mensaje y log con el contenido de `prueba-003`. | Activación automática y lectura del mensaje concreto por Lambda. |

Evidencia destacada de la integración:

```text
2026-11-03 17:16:03 INFO [lambda] ESM: Lambda tics400-validacion-lambda processed 1 SQS messages from tics400-validacion-sqs
TICS400_SQS_RECIBIDO=call_id=prueba-003;bucket=tics400-validacion-s3
```

La primera consulta de logs no mostró la línea. La consulta posterior con `filter-log-events` y los logs del contenedor sí la recuperaron. La función solo leyó e imprimió el mensaje sintético; no descargó ni transcribió el archivo referenciado.


# 10. Soportes y pendientes

La documentación consolida las decisiones tomadas: Lambda, reglas configurables, sin sentimiento, un tipo y un tema por llamada, ETL diario y escenario multiempresa por concentración de llegadas. No deben reabrirse como alternativas pendientes en las diapositivas.

| Pendiente real | Qué debe cerrarse |
|---|---|
| Configuración del procesamiento | Modelo y tamaño de segmentos; memoria, timeout, concurrencia y recomposición. |
| Reglas de análisis | Puntuación, desempate, términos excluidos, protección de información personal y criterios de evaluación. |
| Parámetros operacionales | Aprobar retenciones y objetivo de recuperación; fijar horario del ETL y parámetros restantes. |
| IAM y conectividad | Completar parámetros y coherencia con los diagramas del equipo; verificar controles no probados. |
| Soportes del repositorio | Publicar documento, diagramas, políticas, referencias y evidencia real. |

La presentación conserva las diapositivas 1–10 del equipo. El cierre acordado es: **11 consistencia; 12 identidad y acceso; 13 permisos por componente; 14 JSON; 15 validación; 16 referencias**. Los diagramas de arquitectura y red existentes no se sustituyen en esta actualización. Las diferencias que detecte el equipo se corrigen en sus fuentes, sin afirmar que un esquema haya sido desplegado.

# Referencias

Los identificadores conservan los de la bibliografía consolidada del proyecto. Las fuentes respaldan conceptos y decisiones; **las pruebas locales se respaldan con las salidas compartidas por la integrante, no con la documentación de proveedores**. Esta actualización no realiza una investigación externa nueva.

## Arquitectura, resiliencia y seguridad — AWS

- **[A1]** [Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html).
- **[A2]** [Creating and configuring an Amazon SQS event source mapping](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html).
- **[A3]** [DynamoDB condition expressions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html).
- **[A4]** [Lambda execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html).
- **[A5]** [API Gateway JWT authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-jwt-authorizer.html).
- **[A6]** [Cognito resource servers, scopes and M2M](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html).
- **[A7]** [Cognito authorization endpoint: código y PKCE](https://docs.aws.amazon.com/cognito/latest/developerguide/authorization-endpoint.html).
- **[A8]** [S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html).
- **[A9]** [Glue schedules](https://docs.aws.amazon.com/glue/latest/dg/monitor-data-warehouse-schedule.html).
- **[A10]** [Athena IAM](https://docs.aws.amazon.com/athena/latest/ug/security-iam-athena.html).
- **[A11]** [Athena actions and resources](https://docs.aws.amazon.com/service-authorization/latest/reference/list_athena.html).
- **[A12]** [Athena query results](https://docs.aws.amazon.com/athena/latest/ug/querying.html).
- **[A13]** [Permissions for S3 event notification destinations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/grant-destinations-permissions-to-s3.html).
- **[A14]** [API Gateway permissions to invoke Lambda](https://docs.aws.amazon.com/lambda/latest/dg/services-apigateway.html).
- **[A15]** [Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html).
- **[A16]** [Using dead-letter queues in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html).
- **[A17]** [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html).
- **[A18]** [IAM Access Analyzer policy validation](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-policy-validation.html). Herramienta mencionada como apoyo; no ejecutada en las pruebas registradas.

## Transcripción y simulador

- **[T1]** SYSTRAN y colaboradores. [faster-whisper](https://github.com/SYSTRAN/faster-whisper). Sus benchmarks no son mediciones de este proyecto.
- **[T2]** Ministack. [Services](https://www.ministack.org/docs/services/).
- **[T3]** Ministack. [Repositorio oficial](https://github.com/ministackorg/ministack).

## Escenario y audio

- **[D1]** Residential Tenancies Authority. (2024). [*Annual Report 2023–24*](https://www.rta.qld.gov.au/sites/default/files/2025-09/RTA-Annual-Report-2023-24.pdf), pp. 14–15.
- **[D2]** AWS. [Amazon Connect — Recording ingestion](https://docs.aws.amazon.com/connect/latest/adminguide/recording-ingestion.html). Solo referencia de formato de audio, no un servicio incorporado al proyecto.
- **[D3]** Microsoft. [WAVEFORMATEX structure](https://learn.microsoft.com/en-us/windows/win32/api/mmeapi/ns-mmeapi-waveformatex).
- **[D4]** Microsoft. [Resource Interchange File Format — RIFF](https://learn.microsoft.com/en-us/windows/win32/xaudio2/resource-interchange-file-format--riff-).

## Apoyo metodológico

- **[M1]** Microsoft. [Queue-Based Load Leveling pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling).
- **[M2]** AWS. [PERF05-BP04: Load test your workload](https://docs.aws.amazon.com/wellarchitected/latest/framework/perf_process_culture_load_test.html).
- **[M3]** Google Cloud. [Standard reports: Agents and teams data set](https://docs.cloud.google.com/contact-center/ccai-platform/docs/standard-reports-agents-and-teams-data-set). Distinción de indicadores; no fuente de duración de nuestros archivos.