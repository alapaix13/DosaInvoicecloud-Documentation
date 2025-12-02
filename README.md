# E33 Example

```json
{
  "Encabezado": {
    "Version": "1.0",
    "IdDoc": {
      "TipoeCF": "33",
      "encf": "E330000000001",
      "IndicadorNotaCredito": 0,
      "IndicadorMontoGravado": 0,
      "TipoIngresos": "01",
      "TipoPago": 1
    },
    "Emisor": {
      "RNCEmisor": "133306001",
      "RazonSocialEmisor": "Dosa Systems S.R.L",
      "NombreComercial": "Dosa Systems S.R.L",
      "DireccionEmisor": "Av. Bolivar #834, La Esperilla, 11010 Santo Domingo.",
      "TablaTelefonoEmisor": {
        "TelefonoEmisor": [
          "829-491-0767"
        ]
      },
      "CorreoEmisor": "alejandrolapaixmatos@gmail.com",
      "NumeroFacturaInterna": "IC2507-0001",
      "FechaEmision": "18-07-2025"
    },
    "Comprador": {
      "RNCComprador": "131219772",
      "RazonSocialComprador": "CCAG GROUP",
      "FechaEntrega": "18-07-2025",
      "CodigoInternoComprador": "CU2505-00004"
    },
    "Totales": {
      "MontoGravadoTotal": 180,
      "MontoGravadoI1": 180,
      "MontoGravadoI2": 0,
      "MontoGravadoI3": 0,
      "ITBIS1": 18,
      "ITBIS2": 16,
      "ITBIS3": 0,
      "TotalITBIS": 32.4,
      "TotalITBIS1": 32.4,
      "TotalITBIS2": 0,
      "TotalITBIS3": 0,
      "MontoImpuestoAdicional": 0,
      "MontoTotal": 212.4
    }
  },
  "DetallesItems": {
    "Items": [
      {
        "NumeroLinea": 1,
        "IndicadorFacturacion": 1,
        "NombreItem": "S.R.F",
        "IndicadorBienoServicio": 2,
        "CantidadItem": 1,
        "UnidadMedida": 1,
        "PrecioUnitarioItem": 180,
        "MontoItem": 180
      }
    ]
  },
  "Referencia": {
    "NCFModificado": "E310000000003",
    "FechaNCFModificado": "18-07-2025",
    "CodigoModificacion": 1,
    "RazonModificacion": "Corrección de factura"
  },
  "Totales": null
}
```