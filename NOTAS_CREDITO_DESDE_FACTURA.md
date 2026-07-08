# Endpoint para Crear Notas de Credito desde JSON de Factura

## Descripcion General

Este endpoint permite enviar el JSON de una factura ya emitida para transformarlo y emitirlo como una Nota de Credito electronica E34. El servicio toma la factura original como base, reemplaza los datos propios de la nota, agrega el bloque de referencia requerido por DGII y reutiliza el flujo normal de firma, envio y consulta de estado.

**URL:** `https://dosasystem.com/fe/api/emision/ecf/NotaDesdeFactura`  
**Metodo:** POST  
**Content-Type:** `application/json`

> Nota: el endpoint tambien soporta Nota de Debito E33 usando `tipoeCF=33`, pero esta guia documenta el uso principal para crear Notas de Credito E34.

---

## Autenticacion

Enviar la API Key en el header:

```http
X-API-KEY: TU_API_KEY
```

Si el header no se envia, la API responde `401 Unauthorized`.

---

## Query Params

| Parametro | Tipo | Requerido | Default | Descripcion |
|-----------|------|-----------|---------|-------------|
| `nuevoENCF` | string | Si | N/A | eNCF que se asignara a la nota. Para Nota de Credito debe iniciar con `E34` y tener 13 caracteres. Ejemplo: `E340000000123`. |
| `tipoeCF` | string | No | `34` | Tipo de nota a generar. `34` = Nota de Credito, `33` = Nota de Debito. |
| `codigoModificacion` | integer | No | `1` | Codigo DGII de modificacion. Usar `1` para anulacion, `2` para correccion de texto, `3` para correccion de montos, segun aplique. |
| `razonModificacion` | string | No | `Anula el NCF modificado` | Texto que explica la razon de la nota. |
| `fechaVencimientoSecuencia` | string | Condicional | Se conserva la original | Formato `dd-MM-yyyy`. Solo aplica para `tipoeCF=33`; no se usa para E34. |
| `fechaEmision` | string | No | Fecha actual del servidor | Fecha de emision de la nota en formato `dd-MM-yyyy`. |

---

## Request Body

El cuerpo debe ser el JSON completo de la factura original, con la misma estructura usada por `EnviarFactura`.

Campos minimos importantes del JSON original:

| Campo | Requerido | Descripcion |
|-------|-----------|-------------|
| `encabezado.version` | Si | Version del formato, normalmente `1.0`. |
| `encabezado.idDoc.encf` | Si | eNCF de la factura original que sera modificada. |
| `encabezado.emisor.fechaEmision` | Si | Fecha de emision de la factura original. Se usa como `FechaNCFModificado`. |
| `encabezado.emisor.rncEmisor` | Si | RNC del emisor. |
| `encabezado.totales.montoTotal` | Si | Total de la nota/factura enviada. |
| `detallesItems.items` | Si | Items incluidos en la nota. Ajustar cantidades y montos si la nota es parcial. |

Para una Nota de Credito parcial, el integrador debe enviar el JSON con los montos e items que realmente se van a acreditar. El endpoint no calcula automaticamente una devolucion parcial; transforma y emite el JSON recibido.

---

## Comportamiento del Endpoint

Al recibir la factura original, el API realiza estas transformaciones antes de emitir:

- Cambia `encabezado.idDoc.tipoeCF` a `34`.
- Cambia `encabezado.idDoc.encf` al valor de `nuevoENCF`.
- Elimina campos que DGII no acepta en una nota, como `terminoPago`, `tablaFormasPago`, `indicadorEnvioDiferido`, `indicadorServicioTodoIncluido` y `totalPaginas`.
- Para E34 elimina `fechaVencimientoSecuencia`.
- Calcula `indicadorNotaCredito`: `1` si la nota se emite 30 dias o mas despues de la factura original; `0` si no.
- Actualiza `encabezado.emisor.fechaEmision` con la fecha de la nota.
- Actualiza `encabezado.emisor.numeroFacturaInterna` con `nuevoENCF`.
- Limpia campos no aplicables del comprador para notas, como correo, direccion y orden de compra.
- Agrega el bloque `referencia` con el eNCF original, fecha original, codigo y razon de modificacion.
- Ejecuta el mismo flujo de `EnviarFactura`: validacion, firma XML, envio DGII, persistencia y consulta de estado.

---

## Ejemplo de Request

```http
POST https://dosasystem.com/fe/api/emision/ecf/NotaDesdeFactura?nuevoENCF=E340000000123&tipoeCF=34&codigoModificacion=3&razonModificacion=Devolucion%20parcial%20por%20mercancia%20defectuosa&fechaEmision=18-05-2025
Content-Type: application/json
X-API-KEY: TU_API_KEY
```

```json
{
  "encabezado": {
    "version": "1.0",
    "idDoc": {
      "tipoeCF": "31",
      "encf": "E310000000001",
      "indicadorMontoGravado": 1,
      "tipoIngresos": "01",
      "tipoPago": 2,
      "fechaLimitePago": "15-06-2025",
      "terminoPago": "30 DIAS",
      "fechavencimientosecuencia": "31-12-2025",
      "tablaFormasPago": {
        "formaDePago": [
          {
            "formaPago": "2",
            "montoPago": 118000.00
          }
        ]
      }
    },
    "emisor": {
      "rncEmisor": "101742186",
      "razonSocialEmisor": "Empresa Ejemplo SRL",
      "nombreComercial": "Empresa Ejemplo",
      "direccionEmisor": "Calle Principal No. 123, Santo Domingo",
      "numeroFacturaInterna": "FAC-2025-001",
      "zonaVenta": "PRINCIPAL",
      "fechaEmision": "15-05-2025"
    },
    "comprador": {
      "RNCComprador": "130763102",
      "razonSocialComprador": "Cliente Comercial SA",
      "correoComprador": "facturacion@cliente.com",
      "direccionComprador": "Av. Principal No. 456, Santo Domingo"
    },
    "totales": {
      "MontoGravadoTotal": 50000.00,
      "MontoGravadoI1": 50000.00,
      "MontoGravadoI2": 0.00,
      "MontoGravadoI3": 0.00,
      "MontoExento": 0.00,
      "ITBIS1": 18,
      "ITBIS2": 16,
      "ITBIS3": 0,
      "TotalITBIS": 9000.00,
      "TotalITBIS1": 9000.00,
      "TotalITBIS2": 0.00,
      "TotalITBIS3": 0.00,
      "MontoImpuestoAdicional": 0.00,
      "MontoDescuento": null,
      "MontoRecargo": null,
      "MontoTotal": 59000.00
    }
  },
  "detallesItems": {
    "items": [
      {
        "numeroLinea": 1,
        "indicadorFacturacion": 1,
        "nombreItem": "Devolucion de 1 Laptop Dell Inspiron 15",
        "indicadorBienoServicio": 2,
        "cantidadItem": 1.00,
        "unidadMedida": 1,
        "precioUnitarioItem": 50000.00,
        "montoItem": 50000.00
      }
    ]
  },
  "descuentosORecargos": {
    "descuentoORecargo": []
  },
  "paginacion": {}
}
```

---

## JSON Transformado Internamente

Antes de firmar y enviar a DGII, el endpoint agrega una referencia similar a esta:

```json
{
  "encabezado": {
    "idDoc": {
      "tipoeCF": "34",
      "encf": "E340000000123",
      "indicadorNotaCredito": 0,
      "indicadorMontoGravado": 1,
      "tipoIngresos": "01",
      "tipoPago": 2
    },
    "emisor": {
      "rncEmisor": "101742186",
      "razonSocialEmisor": "Empresa Ejemplo SRL",
      "direccionEmisor": "Calle Principal No. 123, Santo Domingo",
      "numeroFacturaInterna": "E340000000123",
      "fechaEmision": "18-05-2025"
    },
    "comprador": {
      "RNCComprador": "130763102",
      "razonSocialComprador": "Cliente Comercial SA",
      "fechaEntrega": "18-05-2025"
    }
  },
  "referencia": {
    "NCFModificado": "E310000000001",
    "FechaNCFModificado": "15-05-2025",
    "CodigoModificacion": 3,
    "RazonModificacion": "Devolucion parcial por mercancia defectuosa"
  }
}
```

---

## Ejemplo cURL

```bash
curl -X POST "https://dosasystem.com/fe/api/emision/ecf/NotaDesdeFactura?nuevoENCF=E340000000123&tipoeCF=34&codigoModificacion=3&razonModificacion=Devolucion%20parcial%20por%20mercancia%20defectuosa&fechaEmision=18-05-2025" \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: TU_API_KEY" \
  --data @factura-original-ajustada.json
```

---

## Response

La respuesta es la misma estructura que devuelve el flujo de `EnviarFactura`, porque el endpoint reutiliza ese proceso.

Ejemplo de respuesta aceptada o aceptada condicional:

```json
{
  "trackId": "20250518123456abcdef",
  "codigo": 1,
  "estado": "Aceptado",
  "rnc": "101742186",
  "encf": "E340000000123",
  "url": "https://ecf.dgii.gov.do/...",
  "codigoSeguridad": "ABC123",
  "fechaFirma": "18-05-2025 14:35:20",
  "secuenciaUtilizada": true,
  "fechaRecepcion": "2025-05-18T14:35:25",
  "mensajes": []
}
```

---

## Errores Comunes

| Codigo | Mensaje / Causa | Solucion |
|--------|------------------|----------|
| 400 | `Debe indicar 'nuevoENCF'` | Enviar el query param `nuevoENCF`. |
| 400 | `tipoeCF debe ser 34 o 33` | Usar `tipoeCF=34` para Nota de Credito. |
| 400 | `La factura original no tiene eNCF` | Verificar `encabezado.idDoc.encf` en el body. |
| 400 | `El nodo Referencia es obligatorio para E33 y E34` | Normalmente no debe ocurrir en este endpoint; si ocurre, validar que se este llamando `NotaDesdeFactura` y no `EnviarFactura` directamente. |
| 401 | API Key faltante o invalida | Enviar `X-API-KEY` valido. |
| 404 | Configuracion o certificado no encontrado | Revisar configuracion del emisor y certificado digital. |
| 500 | Error al obtener token, firmar o enviar | Revisar logs del backend y respuesta DGII. |

---

## Reglas Practicas para Integradores

- Usar `nuevoENCF` de una secuencia E34 valida y disponible.
- Enviar en el body los montos exactos de la nota. Para devolucion parcial, no enviar la factura completa si solo se acreditara una parte.
- Mantener `encabezado.idDoc.encf` con el eNCF original; el endpoint lo usa para crear `referencia.NCFModificado`.
- Mantener `encabezado.emisor.fechaEmision` con la fecha original; el endpoint la usa como `referencia.FechaNCFModificado`.
- Pasar `fechaEmision` cuando se quiera controlar la fecha exacta de la nota.
- Usar `codigoModificacion=3` cuando la nota corrige montos; usar `1` cuando anula completamente el comprobante.
