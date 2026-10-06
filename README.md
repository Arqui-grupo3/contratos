# Contratos EnergyShark E1 - Grupo 3 (REE)

Esquemas JSON Schema Draft 2020-12 que describen las validaciones actuales del conector.
Este directorio es un repositorio Git independiente con `origin` configurado en
`https://github.com/Arqui-grupo3/contratos`. Los archivos creados localmente deben
versionarse y publicarse para que el resto del equipo pueda consultarlos en GitHub.

## Archivos y uso

| Archivo en `schemas/` | Uso |
| --- | --- |
| `common.schema.json` | Definiciones reutilizables: UUID, texto, fechas, numeros y campos del envelope. |
| `envelope.schema.json` | Selecciona uno de los 12 tipos de mensaje mediante `oneOf`. |
| `central-message.schema.json` | Entrada recomendada para recepcion: envelope completo y `sender: "central"`. |
| `city-message.schema.json` | Entrada recomendada para publicacion de REE: envelope completo y `cityId: "REE"`. |
| `request.schema.json` | `data.ask` como texto no vacio. |
| `ack.schema.json` | `data.target` como UUID. |
| `nack.schema.json` | Motivos/codigos del protocolo y mensaje de rechazo. |
| `error.schema.json` | Errores de negocio y campos adicionales segun el motivo. |
| `status-statement.schema.json` | Estado energetico y fecha `validUntil`. |
| `transfer.schema.json` | Cantidad con cualquier signo y UUID `becauseOf` opcional. |
| `distance-table.schema.json` | Rutas, distancia, costo y booleano `enabled`. |
| `demand-statement.schema.json` | Cantidad con cualquier signo y valor por kWh no negativo. |
| `negotiation-proposal.schema.json` | Direccion give/take, cantidad positiva y precio no negativo. |
| `give.schema.json`, `take.schema.json` | UUID de propuesta, energia positiva y precio no negativo. |
| `negotiation-report.schema.json` | Saldos numericos de presupuesto y energia. |

`examples/messages.json` contiene un ejemplo completo por tipo para validar recepcion.
Para probar publicacion, quitar `sender` y agregar `cityId: "REE"`.
Los esquemas de cada tipo validan el envelope y su contenido; las dos entradas
central/city agregan las condiciones de identidad correspondientes.

Los `$ref` son relativos. Los `$id` identifican los esquemas: no es necesario
consultar GitHub si se cargan los 16 archivos en el validador.

## Reglas conservadas del codigo

- Se requieren `idpk`, `msgId`, `type`, `timestamp` y `data`.
- UUID con guiones, versiones 1-8 y variante RFC; acepta mayusculas/minusculas.
- Texto con al menos un caracter distinto de espacios. No se convierten textos a numeros.
- `data` y objetos internos rechazan `null` y arreglos.
- Numeros no negativos permiten cero; numeros positivos exigen un valor mayor que cero.
- `transfer.quantity`, `balance.quantity` y los saldos del reporte permiten ambos signos y cero.
- `cycleId` no vacio se exige en status-statement, transfer, demand-statement,
  negotiation-proposal, give, take, negotiation-report y error.
- En los demas tipos, un `cycleId` adicional no se restringe, igual que en el codigo.
- Se permiten campos adicionales. `distances` puede estar vacio; no se impone una lista de ciudades.
- `becauseOf` es opcional; cuando aparece debe tener formato UUID.
- Un NACK MALFORMED_MESSAGE exige que `data.target` exista, pero permite cualquier valor
  para devolver un `msgId` mal formado. Los demas NACK exigen un target UUID.
- Un `data.cycleId` adicional de NACK puede conservar el valor original sin validarlo.

| NACK reason | code |
| --- | --- |
| MALFORMED_MESSAGE | 422 |
| UNKNOWN_TYPE | 400 |
| IDPK_EQUALS_MSGID | 422 |
| IDENTITY_MISMATCH | 403 |

| Error reason | code | Campo adicional de data |
| --- | --- | --- |
| CYCLE_UNKNOWN | 404 | Ninguno |
| CYCLE_EXPIRED | 410 | Ninguno |
| PRICE_ABOVE_CAP | 422 | cap no negativo |
| OVER_CAPACITY | 409 | spare no negativo |
| REPORT_TOO_EARLY | 422 o 425 | opensAt como fecha ISO 8601 |

Se conservan los dos codigos de REPORT_TOO_EARLY que acepta el conector por la
inconsistencia del enunciado. PRICE_ABOVE_CAP pertenece a error, segun el texto
normativo y la implementacion, aunque un ejemplo del enunciado lo etiqueta nack.

## Reglas que se verifican en codigo

JSON Schema estandar no compara los valores de dos propiedades. Despues de validar
el esquema, se debe comprobar que `idpk.toLowerCase() !== msgId.toLowerCase()`.
El conector ya lo hace; el publicador genera un msgId nuevo y diferente.

Las fechas usan patron ISO 8601 y `format: "date-time"`. Algunos validadores no
aplican format por defecto: debe habilitarse. El conector comprueba tambien
que la fecha y hora existan mediante `isTimestamp`.

JSON solo representa numeros finitos. Cuando se valida directamente un objeto de
JavaScript, el validador debe rechazar NaN e Infinity, como hace el conector.

AMQP `user_id = city.REE`, exchange, routing key, TLS y ausencia de declaraciones
de topologia son condiciones del transporte. No se agregan como campos del JSON.
`sender: "central"` comprueba el valor declarado en el mensaje; no autentica por si solo al emisor.

Los esquemas verifican estructura y tipos. La existencia/vigencia del ciclo,
referencias a mensajes, saldo disponible, precios maximos y persistencia requieren
la logica correspondiente del sistema.

## Verificacion realizada

Antes de retirar los scripts y archivos temporales, se realizaron 322 comprobaciones
locales con el validador de PowerShell y se confirmo su concordancia con el conector.
La prueba de conexion y procesamiento con el broker real sigue pendiente.
