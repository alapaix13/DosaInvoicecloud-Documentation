# Endpoint de Envío de Facturas Electrónicas (eCF)

## Documentacion Adicional

- [Endpoint para Crear Notas de Credito desde JSON de Factura](NOTAS_CREDITO_DESDE_FACTURA.md)

---

## Correcciones para quien ya integró (noviembre 2026)

Versiones anteriores de este documento tenían códigos que no son los de la DGII. Si su integración se hizo con ellas, revise estos puntos:

| Dato | Decía antes | Lo correcto |
| --- | --- | --- |
| `indicadorFacturacion` | 1 producto, 2 servicio, 3 mixta, 4 profesional | 1 = ITBIS 18 %, 2 = ITBIS 16 %, 3 = ITBIS 0 %, 4 = exento. No dice si es producto o servicio. |
| `indicadorBienoServicio` | 1 servicio, 2 bien | 1 = bien, 2 = servicio |
| `unidadMedida` | 1 = unidad, y una tabla de 20 códigos | 43 = unidad, 1 = barril, y otra tabla. Ver [equivalencias](#catálogo-de-unidades-de-medida). Es opcional. |
| `formaPago` | 3 transferencia, 4 tarjeta de crédito, 5 tarjeta de débito, 8 bonos, 9 otras | 2 transferencia, 3 tarjeta (débito o crédito), 4 venta a crédito, 5 bonos, 8 otras. Ver [equivalencias](#catálogo-de-formas-de-pago). |
| Secuencia del e-NCF | 11 dígitos | 10 dígitos: 13 caracteres en total |
| Comprobante rechazado | Un ejemplo de código lo trataba como `codigo` 0 | `codigo` 2. Lo más seguro es decidir por el texto de `estado`. |
| Reintentos | «Nunca reutilizar una secuencia» y «reintente con el mismo eNCF» a la vez | Ver [qué hacer según la respuesta](#2-manejo-de-errores-y-reintentos) |
| Ejemplos E32 (con compra), E41, E43, E44, E45 y E46 | El indicador de la línea no cuadraba con los totales | Corregidos. El E46 ahora va a tasa 0 % (`MontoGravadoI3`): su XSD no admite `MontoExento`. |

Un comprobante ya aceptado por la DGII no se modifica. Si emitió con códigos equivocados, revise con su contador si hace falta corregirlos con una nota de crédito y cómo afecta sus reportes.

---

## Descripción General

Este endpoint permite el envío de Comprobantes Fiscales Electrónicos (eCF) al sistema de facturación electrónica de la República Dominicana. El servicio procesa diferentes tipos de documentos fiscales según las normativas de la DGII (Dirección General de Impuestos Internos).

**URL:** `https://dosasystem.com/fe/api/emision/ecf/EnviarFactura`  
**Método:** POST  
**Content-Type:** application/json

---

## Autenticación

El endpoint requiere autenticación mediante API Key en los headers:

```
X-API-KEY: [Tu API Key]

 ```

---

## Estructura General del Request Body

El cuerpo de la solicitud está compuesto por los siguientes bloques principales:

### 1\. **encabezado**

Contiene la información general del documento fiscal.

#### **encabezado.version**

- **Tipo:** String
    
- **Requerido:** Sí
    
- **Descripción:** Versión del formato de eCF (actualmente "1.0")
    

#### **encabezado.idDoc** (Identificación del Documento)

Información que identifica el tipo y características del comprobante:

- **tipoeCF** (String, Requerido): Código del tipo de comprobante fiscal electrónico
    
    - `31`: Factura de Crédito Fiscal (E31)
        
    - `32`: Factura de Consumo (E32)
        
    - `33`: Nota de Débito (E33)
        
    - `34`: Nota de Crédito (E34)
        
    - `41`: Compras (E41)
        
    - `43`: Comprobante de Gastos Menores (E43)
        
    - `44`: Regímenes Especiales (E44)
        
    - `45`: Gubernamental (E45)
        
    - `46`: Exportaciones (E46)
        
    - `47`: Comprobante de Pagos al Exterior de Servicios (E47)
        
- **encf** (String, Requerido): Número del comprobante fiscal electrónico. 13 caracteres: `E` + 2 dígitos del tipo + 10 dígitos de secuencia (ejemplo: `E310000000001`)
    
    - Ejemplo: "E320000000004"
        
- **indicadorEnvioDiferido** (Integer, Opcional): Indica si el envío es diferido
    
    - `null` o no incluido: Envío inmediato
        
    - `1`: Envío diferido
        
- **indicadorMontoGravado** (Integer, Requerido): Indica si el monto está gravado
    
    - `0`: No gravado
        
    - `1`: Gravado
        
- **indicadorServicioTodoIncluido** (Integer, Opcional): Para servicios todo incluido
    
    - `null`: No aplica
        
    - `1`: Servicio todo incluido
        
- **tipoIngresos** (String, Requerido): Tipo de ingreso
    
    - `01`: Ingresos por operaciones (No financieros)
        
    - `02`: Ingresos financieros
        
    - `03`: Ingresos extraordinarios
        
    - `04`: Ingresos por arrendamientos
        
    - `05`: Ingresos por venta de activo depreciable
        
    - `06`: Otros ingresos
        
- **tipoPago** (Integer, Requerido): Tipo de pago (contado, crédito o gratuito). No confundir con `formaPago` (efectivo, transferencia, tarjeta…), que va en `tablaFormasPago`
    
    - `1`: Contado
        
    - `2`: Crédito
        
    - `3`: Gratuito
        
- **fechaLimitePago** (String, Condicional): Fecha límite de pago (formato: DD-MM-YYYY)
    
    - Requerido cuando tipoPago = 2 (Crédito)
        
- **terminoPago** (String, Opcional): Descripción del término de pago
    
    - Ejemplo: "CONTADO", "30 DIAS", "60 DIAS"
        
- **fechavencimientosecuencia** (String, Condicional): Fecha de vencimiento de la secuencia (formato: DD-MM-YYYY)
    
    - **Requerido para todos los tipos de eCF excepto E32 y E34**
        
    - No se debe incluir para tipos E32 (Factura de Consumo) y E34 (Nota de Crédito)
        
    - Debe ser una fecha válida en el futuro
        
- **tablaFormasPago** (Object, Requerido): Detalle de las formas de pago utilizadas
    
    - **formaDePago** (Array): Lista de formas de pago
        
        - **formaPago** (String): Código de forma de pago de la DGII (ver [Catálogo de Formas de Pago](#catálogo-de-formas-de-pago)): `1` efectivo, `2` cheque/transferencia/depósito, `3` tarjeta de débito o crédito, `4` venta a crédito
            
        - **montoPago** (Decimal): Monto pagado con esta forma
            

#### **encabezado.emisor** (Datos del Emisor)

Información del emisor del comprobante:

- **rncEmisor** (String, Requerido): RNC del emisor (9 u 11 dígitos)
    
- **razonSocialEmisor** (String, Requerido): Razón social del emisor
    
- **nombreComercial** (String, Requerido): Nombre comercial del emisor
    
- **direccionEmisor** (String, Requerido): Dirección completa del emisor
    
- **numeroFacturaInterna** (String, Opcional): Número de factura interno del sistema
    
- **zonaVenta** (String, Opcional): Zona de venta o sucursal
    
- **fechaEmision** (String, Requerido): Fecha de emisión (formato: DD-MM-YYYY)
    

#### **encabezado.comprador** (Datos del Comprador)

Información del comprador o receptor del comprobante:

- **RNCComprador** (String, Condicional): RNC del comprador (requerido para E31, E41, E44)
    
- **identificadorextranjero** (String, Condicional): Identificador para compradores extranjeros
    
- **razonSocialComprador** (String, Requerido): Razón social o nombre del comprador
    
- **correoComprador** (String, Opcional): Correo electrónico del comprador
    
- **direccionComprador** (String, Opcional): Dirección del comprador
    
- **fechaEntrega** (String, Opcional): Fecha de entrega (formato: DD-MM-YYYY)
    
- **fechaOrdenCompra** (String, Opcional): Fecha de orden de compra
    
- **numeroOrdenCompra** (String, Opcional): Número de orden de compra
    
- **codigoInternoComprador** (String, Opcional): Código interno del comprador
    

#### **encabezado.totales** (Totales del Documento)

Resumen de montos del comprobante:

- **MontoGravadoTotal** (Decimal, Requerido): Suma de todos los montos gravados
    
- **MontoGravadoI1** (Decimal): Monto gravado con tasa ITBIS 1 (18%)
    
- **MontoGravadoI2** (Decimal): Monto gravado con tasa ITBIS 2 (16%)
    
- **MontoGravadoI3** (Decimal): Monto gravado con tasa ITBIS 3 (0%)
    
- **MontoExento** (Decimal, Requerido): Monto exento de impuestos
    
- **ITBIS1** (Decimal): Tasa de ITBIS 1 (generalmente 18)
    
- **ITBIS2** (Decimal): Tasa de ITBIS 2 (generalmente 16)
    
- **ITBIS3** (Decimal): Tasa de ITBIS 3 (generalmente 0)
    
- **TotalITBIS** (Decimal, Requerido): Total de ITBIS calculado
    
- **TotalITBIS1** (Decimal): ITBIS calculado con tasa 1
    
- **TotalITBIS2** (Decimal): ITBIS calculado con tasa 2
    
- **TotalITBIS3** (Decimal): ITBIS calculado con tasa 3
    
- **MontoImpuestoAdicional** (Decimal): Monto de impuestos adicionales
    
- **MontoDescuento** (Decimal, Opcional): Monto total de descuentos
    
- **MontoRecargo** (Decimal, Opcional): Monto total de recargos
    
- **MontoTotal** (Decimal, Requerido): Monto total del comprobante
    

#### **encabezado.OtraMoneda** (Información en Otra Moneda)

Información cuando la transacción se realiza en moneda extranjera:

- **TipoMoneda** (String, Requerido): Código de moneda (USD, EUR, etc.)
    
- **TipoCambio** (Decimal, Requerido): Tasa de cambio aplicada
    
- **MontoExentoOtraMoneda** (Decimal): Monto exento en otra moneda
    
- **MontoGravado1OtraMoneda** (Decimal): Monto gravado tasa 1 en otra moneda
    
- **MontoGravado2OtraMoneda** (Decimal): Monto gravado tasa 2 en otra moneda
    
- **MontoGravado3OtraMoneda** (Decimal): Monto gravado tasa 3 en otra moneda
    
- **MontoGravadoTotalOtraMoneda** (Decimal): Total gravado en otra moneda
    
- **TotalITBISOtraMoneda** (Decimal): Total ITBIS en otra moneda
    
- **TotalITBIS1OtraMoneda** (Decimal): ITBIS tasa 1 en otra moneda
    
- **TotalITBIS2OtraMoneda** (Decimal): ITBIS tasa 2 en otra moneda
    
- **TotalITBIS3OtraMoneda** (Decimal): ITBIS tasa 3 en otra moneda
    
- **MontoImpuestoAdicionalOtraMoneda** (Decimal): Impuestos adicionales en otra moneda
    
- **MontoDescuentoOtraMoneda** (Decimal): Descuentos en otra moneda
    
- **MontoRecargoOtraMoneda** (Decimal): Recargos en otra moneda
    
- **MontoTotalOtraMoneda** (Decimal): Monto total en otra moneda
    

### 2\. **detallesItems**

Detalle de los productos o servicios incluidos en el comprobante.

#### **detallesItems.items** (Array)

Lista de ítems del comprobante:

- **numeroLinea** (Integer, Requerido): Número secuencial de la línea
    
- **indicadorFacturacion** (Integer, Requerido): Cómo se grava la línea (ver [catálogo](#catálogo-de-indicadores-de-facturación))
    
    - `1`: ITBIS 18 %
        
    - `2`: ITBIS 16 %
        
    - `3`: ITBIS 0 %
        
    - `4`: Exento
        
- **nombreItem** (String, Requerido): Descripción del producto o servicio
    
- **indicadorBienoServicio** (Integer, Requerido): Indica si es bien o servicio
    
    - `1`: Bien
        
    - `2`: Servicio
        
- **cantidadItem** (Decimal, Requerido): Cantidad del ítem
    
- **unidadMedida** (Integer, Opcional): Código de unidad de medida de la DGII (ver [catálogo](#catálogo-de-unidades-de-medida)). Unidad = `43`
    
- **precioUnitarioItem** (Decimal, Requerido): Precio unitario del ítem
    
- **montoItem** (Decimal, Requerido): Monto total del ítem (cantidad × precio)
    
- **tablaImpuestoAdicional** (Object, Opcional): Impuestos adicionales aplicables
    
- **OtraMonedaDetalle** (Object, Condicional): Detalle en otra moneda
    
    - **PrecioOtraMoneda** (Decimal): Precio en otra moneda
        
    - **MontoItemOtraMoneda** (Decimal): Monto del ítem en otra moneda
        

### 3\. **Subtotales**

Subtotales opcionales para agrupación de ítems.

### 4\. **descuentosORecargos**

Descuentos o recargos aplicados al documento.

#### **descuentosORecargos.descuentoORecargo** (Array)

- **tipo** (Integer): Tipo de ajuste (1: Descuento, 2: Recargo)
    
- **descripcion** (String): Descripción del ajuste
    
- **monto** (Decimal): Monto del ajuste
    
- **indicadorNorma** (Integer): Indicador de norma aplicable
    

### 5\. **paginacion**

Información de paginación para documentos extensos.

### 6\. **informacionReferencia**

Referencias a otros documentos fiscales (usado en notas de crédito/débito).

---

## Documentación Detallada por Tipo de eCF

### E31: Factura de Crédito Fiscal

#### **Propósito**

La Factura de Crédito Fiscal (E31) es el comprobante que respalda las transacciones comerciales entre contribuyentes registrados en el RNC. Permite al comprador utilizar el ITBIS pagado como crédito fiscal deducible.

#### **Características Principales**

- Debe emitirse entre contribuyentes registrados
    
- El comprador debe tener RNC válido
    
- Permite deducción del ITBIS como crédito fiscal
    
- Requiere información detallada del comprador
    
- Debe incluir desglose de ITBIS
    

#### **Campos Obligatorios Específicos**

- `encabezado.idDoc.tipoeCF`: "31"
    
- `encabezado.comprador.RNCComprador`: Obligatorio (9 u 11 dígitos)
    
- `encabezado.comprador.razonSocialComprador`: Obligatorio
    
- `encabezado.totales.MontoGravadoTotal`: Debe ser mayor a 0
    
- `encabezado.totales.TotalITBIS`: Debe calcularse correctamente
    

#### **Reglas de Validación**

1. El RNC del comprador debe ser válido y estar registrado en DGII
    
2. El monto gravado debe tener ITBIS calculado correctamente
    
3. La suma de MontoGravadoI1 + MontoGravadoI2 + MontoGravadoI3 debe igual MontoGravadoTotal
    
4. TotalITBIS = (MontoGravadoI1 × ITBIS1/100) + (MontoGravadoI2 × ITBIS2/100) + (MontoGravadoI3 × ITBIS3/100)
    
5. MontoTotal = MontoGravadoTotal + MontoExento + TotalITBIS + MontoImpuestoAdicional - MontoDescuento + MontoRecargo
    
6. **El campo fechavencimientosecuencia es obligatorio para E31**
    

#### **Ejemplo Práctico E31**

``` json
{
  "encabezado": {
    "version": "1.0",
    "idDoc": {
      "tipoeCF": "31",
      "encf": "E310000000001",
      "indicadorEnvioDiferido": null,
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
            "montoPago": "118000.00"
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
      "correoComprador": "facturacion@clientecomercial.com",
      "direccionComprador": "Av. Winston Churchill No. 456, Santo Domingo",
      "codigoInternoComprador": "CLI-001"
    },
    "totales": {
      "MontoGravadoTotal": "100000.00",
      "MontoGravadoI1": "100000.00",
      "MontoGravadoI2": "0.00",
      "MontoGravadoI3": "0.00",
      "MontoExento": "0.00",
      "ITBIS1": 18,
      "ITBIS2": 16,
      "ITBIS3": 0,
      "TotalITBIS": "18000.00",
      "TotalITBIS1": "18000.00",
      "TotalITBIS2": "0.00",
      "TotalITBIS3": "0.00",
      "MontoImpuestoAdicional": "0.00",
      "MontoDescuento": null,
      "MontoRecargo": null,
      "MontoTotal": "118000.00"
    }
  },
  "detallesItems": {
    "items": [
      {
        "numeroLinea": 1,
        "indicadorFacturacion": 1,
        "nombreItem": "Laptop Dell Inspiron 15",
        "indicadorBienoServicio": 1,
        "cantidadItem": "2.00",
        "unidadMedida": 43,
        "precioUnitarioItem": "50000.00",
        "montoItem": "100000.00",
        "tablaImpuestoAdicional": {}
      }
    ]
  },
  "Subtotales": {},
  "descuentosORecargos": {
    "descuentoORecargo": []
  },
  "paginacion": {},
  "informacionReferencia": {}
}

 ```

#### **Casos de Uso Comunes E31**

- Venta de productos entre empresas
    
- Prestación de servicios profesionales a empresas
    
- Venta de equipos y maquinaria
    
- Servicios de consultoría
    
- Suministros industriales
    

---

### E32: Factura de Consumo

#### **Propósito**

La Factura de Consumo (E32) se utiliza para transacciones con consumidores finales o cuando el comprador no requiere crédito fiscal. Es el equivalente electrónico del antiguo comprobante fiscal de consumo.

#### **Características Principales**

- Se emite a consumidores finales
    
- El RNC del comprador es opcional
    
- No permite deducción de ITBIS como crédito fiscal
    
- Puede incluir identificador de extranjero
    
- Información del comprador puede ser simplificada
    

#### **Campos Obligatorios Específicos**

- `encabezado.idDoc.tipoeCF`: "32"
    
- `encabezado.comprador.razonSocialComprador`: Obligatorio (puede ser "CONSUMIDOR FINAL")
    
- `encabezado.comprador.RNCComprador`: Opcional
    
- `encabezado.totales.MontoTotal`: Obligatorio
    

#### **Reglas de Validación**

1. Si el comprador no tiene RNC, puede dejarse vacío o usar "CONSUMIDOR FINAL"
    
2. Para extranjeros, usar campo `identificadorextranjero`
    
3. Los cálculos de ITBIS deben ser correctos aunque no sea deducible
    
4. Puede incluir montos exentos sin restricciones especiales
    
5. **El campo fechavencimientosecuencia NO debe incluirse para E32**
    

#### **Ejemplo Práctico E32**

``` json
{
  "encabezado": {
    "version": "1.0",
    "idDoc": {
      "tipoeCF": "32",
      "encf": "E320000000005",
      "indicadorEnvioDiferido": null,
      "indicadorMontoGravado": 1,
      "tipoIngresos": "01",
      "tipoPago": 1,
      "terminoPago": "CONTADO",
      "tablaFormasPago": {
        "formaDePago": [
          {
            "formaPago": "1",
            "montoPago": "5900.00"
          }
        ]
      }
    },
    "emisor": {
      "rncEmisor": "101742186",
      "razonSocialEmisor": "Tienda Retail SRL",
      "nombreComercial": "Tienda Retail",
      "direccionEmisor": "Plaza Comercial, Local 45, Santo Domingo",
      "numeroFacturaInterna": "VENTA-2025-1234",
      "zonaVenta": "TIENDA-01",
      "fechaEmision": "15-05-2025"
    },
    "comprador": {
      "RNCComprador": "",
      "razonSocialComprador": "CONSUMIDOR FINAL",
      "correoComprador": "cliente@email.com",
      "direccionComprador": "",
      "codigoInternoComprador": ""
    },
    "totales": {
      "MontoGravadoTotal": "5000.00",
      "MontoGravadoI1": "5000.00",
      "MontoGravadoI2": "0.00",
      "MontoGravadoI3": "0.00",
      "MontoExento": "0.00",
      "ITBIS1": 18,
      "ITBIS2": 16,
      "ITBIS3": 0,
      "TotalITBIS": "900.00",
      "TotalITBIS1": "900.00",
      "TotalITBIS2": "0.00",
      "TotalITBIS3": "0.00",
      "MontoImpuestoAdicional": "0.00",
      "MontoDescuento": null,
      "MontoRecargo": null,
      "MontoTotal": "5900.00"
    }
  },
  "detallesItems": {
    "items": [
      {
        "numeroLinea": 1,
        "indicadorFacturacion": 1,
        "nombreItem": "Smartphone Samsung Galaxy A54",
        "indicadorBienoServicio": 1,
        "cantidadItem": "1.00",
        "unidadMedida": 43,
        "precioUnitarioItem": "5000.00",
        "montoItem": "5000.00",
        "tablaImpuestoAdicional": {}
      }
    ]
  },
  "Subtotales": {},
  "descuentosORecargos": {
    "descuentoORecargo": []
  },
  "paginacion": {},
  "informacionReferencia": {}
}

 ```

#### **Ejemplo E32 con Comprador Extranjero**

``` json
{
  "encabezado": {
    "version": "1.0",
    "idDoc": {
      "tipoeCF": "32",
      "encf": "E320000000006",
      "indicadorMontoGravado": 0,
      "tipoIngresos": "01",
      "tipoPago": 1,
      "terminoPago": "CONTADO",
      "tablaFormasPago": {
        "formaDePago": [
          {
            "formaPago": "1",
            "montoPago": "3491052.52"
          }
        ]
      }
    },
    "emisor": {
      "rncEmisor": "101742186",
      "razonSocialEmisor": "Dosa Systems, S.R.L.",
      "nombreComercial": "Dosa Systems",
      "direccionEmisor": "Avenida Pedro Henríquez Ureña No. 150",
      "numeroFacturaInterna": "E320000000001",
      "zonaVenta": "PRINCIPAL",
      "fechaEmision": "02-05-2025"
    },
    "comprador": {
      "RNCComprador": "",
      "identificadorextranjero": "23HH25789",
      "razonSocialComprador": "FRANÇOIS XAVIER DE LA POYPE",
      "correoComprador": "cliente@email.com",
      "direccionComprador": "La Loire No. 00 CP 49125, Cheffes, Francia",
      "fechaEntrega": "02-05-2025",
      "codigoInternoComprador": "CU41"
    },
    "totales": {
      "MontoGravadoTotal": "2958519.03",
      "MontoGravadoI1": "2958519.03",
      "MontoGravadoI2": "0.00",
      "MontoGravadoI3": "0.00",
      "MontoExento": "0.00",
      "ITBIS1": 18,
      "ITBIS2": 16,
      "ITBIS3": 0,
      "TotalITBIS": "532533.50",
      "TotalITBIS1": "532533.50",
      "TotalITBIS2": "0.00",
      "TotalITBIS3": "0.00",
      "MontoImpuestoAdicional": "0.00",
      "MontoTotal": "3491052.52"
    },
    "OtraMoneda": {
      "TipoMoneda": "USD",
      "TipoCambio": "58.86",
      "MontoExentoOtraMoneda": "0.00",
      "MontoGravado1OtraMoneda": "50263.66",
      "MontoGravadoTotalOtraMoneda": "50263.66",
      "TotalITBISOtraMoneda": "9047.46",
      "TotalITBIS1OtraMoneda": "9047.46",
      "MontoTotalOtraMoneda": "59311.12"
    }
  },
  "detallesItems": {
    "items": [
      {
        "numeroLinea": 1,
        "indicadorFacturacion": 1,
        "nombreItem": "SERVICIOS JURIDICOS: RESOLUCIÓN DE CONFLICTOS",
        "indicadorBienoServicio": 2,
        "cantidadItem": "1.00",
        "unidadMedida": 43,
        "precioUnitarioItem": "2958519.03",
        "montoItem": "2958519.03",
        "OtraMonedaDetalle": {
          "PrecioOtraMoneda": "50263.66",
          "MontoItemOtraMoneda": "50263.66"
        }
      }
    ]
  }
}

 ```

#### **Casos de Uso Comunes E32**

- Ventas al detalle en tiendas
    
- Servicios a consumidores finales
    
- Restaurantes y cafeterías
    
- Servicios profesionales a personas físicas
    
- Ventas a turistas extranjeros
    

---

### E33: Nota de Débito

#### **Propósito**

La Nota de Débito (E33) se utiliza para aumentar el valor de una factura previamente emitida. Documenta cargos adicionales, intereses por mora, ajustes de precio, o correcciones que incrementan el monto original.

#### **Características Principales**

- Modifica una factura existente (E31 o E32)
    
- Aumenta el monto de la transacción original
    
- Debe referenciar el documento original
    
- Genera ITBIS adicional si aplica
    
- Requiere justificación del ajuste
    

#### **Campos Obligatorios Específicos**

- `Encabezado.IdDoc.TipoeCF`: "33"
    
- `Referencia`: Obligatorio - debe referenciar el eCF original
    
    - `NCFModificado`: Número del eCF original
        
    - `FechaNCFModificado`: Fecha del eCF original
        
    - `CodigoModificacion`: Código del tipo de modificación
        
    - `RazonModificacion`: Motivo de la nota de débito
        

#### **Reglas de Validación**

1. Debe existir un eCF previo válido para referenciar
    
2. El RNC del comprador debe coincidir con el documento original
    
3. Los montos deben ser positivos (incremento)
    
4. La fecha de emisión debe ser posterior al documento original
    
5. El tipo de moneda debe coincidir con el documento original
    
6. **El campo fechavencimientosecuencia es obligatorio para E33**
    

#### **Ejemplo Práctico E33**

``` json
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
      "NumeroFacturaInterna": "(PROV200)",
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
        "IndicadorBienoServicio": 1,
        "CantidadItem": 1,
        "UnidadMedida": 43,
        "PrecioUnitarioItem": 180,
        "MontoItem": 180
      }
    ]
  },
  "Referencia": {
    "NCFModificado": "E310000000003",
    "FechaNCFModificado": "18-07-2025",
    "CodigoModificacion": 1,
    "RazonModificacion": "Ajuste por diferencia de precio"
  },
  "Totales": null
}

 ```

#### **Motivos Comunes para Notas de Débito**

- Ajuste de precio por error en factura original
    
- Cargos por intereses de mora
    
- Gastos adicionales no incluidos originalmente
    
- Fletes o transportes adicionales
    
- Servicios complementarios agregados
    
- Corrección de cantidades (aumento)
    
- Penalizaciones contractuales
    

---

### E34: Nota de Crédito

#### **Propósito**

La Nota de Crédito (E34) se utiliza para disminuir el valor de una factura previamente emitida. Documenta devoluciones, descuentos posteriores, anulaciones parciales o totales, o correcciones que reducen el monto original.

#### **Características Principales**

- Modifica una factura existente (E31 o E32)
    
- Disminuye el monto de la transacción original
    
- Debe referenciar el documento original
    
- Reduce el ITBIS si aplica
    
- Puede ser parcial o total
    

#### **Campos Obligatorios Específicos**

- `Encabezado.IdDoc.TipoeCF`: "34"
    
- `Referencia`: Obligatorio - debe referenciar el eCF original
    
    - `NCFModificado`: Número del eCF original
        
    - `FechaNCFModificado`: Fecha del eCF original
        
    - `CodigoModificacion`: Código del tipo de modificación
        
    - `RazonModificacion`: Motivo de la nota de crédito
        

#### **Reglas de Validación**

1. Debe existir un eCF previo válido para referenciar
    
2. El RNC del comprador debe coincidir con el documento original
    
3. Los montos deben ser positivos pero representan una reducción
    
4. El monto de la NC no puede exceder el monto del documento original
    
5. La fecha de emisión debe ser posterior al documento original
    
6. Si es anulación total, el monto debe coincidir con el documento original
    
7. **El campo fechavencimientosecuencia NO debe incluirse para E34**
    

#### **Ejemplo Práctico E34 - Devolución Parcial**

``` json
{
  "Encabezado": {
    "Version": "1.0",
    "IdDoc": {
      "TipoeCF": "34",
      "encf": "E340000000001",
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
      "NumeroFacturaInterna": "(PROV200)",
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
        "IndicadorBienoServicio": 1,
        "CantidadItem": 1,
        "UnidadMedida": 43,
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

#### **Ejemplo E34 - Anulación Total**

``` json
{
  "Encabezado": {
    "Version": "1.0",
    "IdDoc": {
      "TipoeCF": "34",
      "encf": "E340000000002",
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
      "NumeroFacturaInterna": "(PROV200)",
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
        "IndicadorBienoServicio": 1,
        "CantidadItem": 1,
        "UnidadMedida": 43,
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

#### **Motivos Comunes para Notas de Crédito**

- Devolución de mercancía
    
- Descuentos otorgados posteriormente
    
- Anulación de factura por error
    
- Bonificaciones comerciales
    
- Ajuste por productos defectuosos
    
- Corrección de cantidades (disminución)
    
- Cancelación de servicios no prestados
    

---

### E41: Compras

#### **Propósito**

El Comprobante de Compras (E41) se utiliza para respaldar las adquisiciones realizadas a proveedores informales o personas físicas que no emiten comprobantes fiscales. Permite al comprador documentar sus gastos y costos deducibles.

#### **Características Principales**

- Emitido por el comprador (no por el vendedor)
    
- Documenta compras a proveedores sin comprobantes
    
- No genera crédito fiscal de ITBIS
    
- Requiere identificación del proveedor
    
- Usado para fines de deducibilidad de gastos
    

#### **Campos Obligatorios Específicos**

- `encabezado.idDoc.tipoeCF`: "41"
    
- `encabezado.comprador`: En este caso representa al vendedor/proveedor
    
    - `RNCComprador`: Cédula o RNC del proveedor (si aplica)
        
    - `razonSocialComprador`: Nombre del proveedor
        
- `encabezado.emisor`: Representa al comprador que emite el E41
    
- `encabezado.totales.MontoExento`: Generalmente todo el monto es exento
    

#### **Reglas de Validación**

1. El emisor del E41 es quien realiza la compra
    
2. El "comprador" en el JSON representa al proveedor/vendedor
    
3. Generalmente no incluye ITBIS deducible
    
4. Debe documentar la naturaleza de la compra
    
5. Límites de monto según normativa DGII
    
6. **El campo fechavencimientosecuencia es obligatorio para E41**
    

#### **Ejemplo Práctico E41**

``` json
{
  "encabezado": {
    "version": "1.0",
    "idDoc": {
      "tipoeCF": "41",
      "encf": "E410000000001",
      "indicadorMontoGravado": 0,
      "tipoIngresos": "01",
      "tipoPago": 1,
      "terminoPago": "CONTADO",
      "fechavencimientosecuencia": "31-12-2025",
      "tablaFormasPago": {
        "formaDePago": [
          {
            "formaPago": "1",
            "montoPago": "15000.00"
          }
        ]
      }
    },
    "emisor": {
      "rncEmisor": "101742186",
      "razonSocialEmisor": "Empresa Compradora SRL",
      "nombreComercial": "Empresa Compradora",
      "direccionEmisor": "Calle Principal No. 123, Santo Domingo",
      "numeroFacturaInterna": "COMP-2025-001",
      "zonaVenta": "COMPRAS",
      "fechaEmision": "15-05-2025"
    },
    "comprador": {
      "RNCComprador": "40212345678",
      "razonSocialComprador": "Juan Pérez Martínez",
      "direccionComprador": "Calle Secundaria No. 45, Santo Domingo",
      "codigoInternoComprador": "PROV-001"
    },
    "totales": {
      "MontoGravadoTotal": "0.00",
      "MontoGravadoI1": "0.00",
      "MontoGravadoI2": "0.00",
      "MontoGravadoI3": "0.00",
      "MontoExento": "15000.00",
      "ITBIS1": 0,
      "ITBIS2": 0,
      "ITBIS3": 0,
      "TotalITBIS": "0.00",
      "TotalITBIS1": "0.00",
      "TotalITBIS2": "0.00",
      "TotalITBIS3": "0.00",
      "MontoTotal": "15000.00"
    }
  },
  "detallesItems": {
    "items": [
      {
        "numeroLinea": 1,
        "indicadorFacturacion": 4,
        "nombreItem": "Compra de productos agrícolas - Plátanos",
        "indicadorBienoServicio": 1,
        "cantidadItem": "500.00",
        "unidadMedida": 21,
        "precioUnitarioItem": "30.00",
        "montoItem": "15000.00"
      }
    ]
  },
  "Subtotales": {},
  "descuentosORecargos": {
    "descuentoORecargo": []
  },
  "paginacion": {},
  "informacionReferencia": {}
}

 ```

#### **Casos de Uso Comunes E41**

- Compras a productores agrícolas
    
- Adquisiciones a artesanos
    
- Compras a personas físicas sin RNC
    
- Servicios de trabajadores independientes informales
    
- Compras menores a proveedores ocasionales
    

---

### E43: Comprobante de Gastos Menores

#### **Propósito**

El Comprobante de Gastos Menores (E43) se utiliza para documentar gastos de montos reducidos realizados a proveedores informales o en situaciones donde no es posible obtener un comprobante fiscal formal. Permite a las empresas justificar gastos operacionales menores como parte de sus costos deducibles.

#### **Características Principales**

- Para gastos de montos menores
    
- Emitido por el comprador (similar a E41)
    
- No requiere RNC del proveedor en muchos casos
    
- Límite de monto establecido por DGII
    
- Documentación simplificada
    
- Generalmente sin ITBIS deducible
    

#### **Campos Obligatorios Específicos**

- `encabezado.idDoc.tipoeCF`: "43"
    
- `encabezado.idDoc.fechavencimientosecuencia`: Obligatorio
    
- `encabezado.emisor`: Representa a la empresa que realiza el gasto
    
- `encabezado.comprador`: Representa al proveedor/vendedor
    
    - `razonSocialComprador`: Nombre del proveedor
        
    - `RNCComprador`: Opcional si es persona informal
        
- `encabezado.totales.MontoTotal`: Debe estar dentro del límite permitido
    

#### **Reglas de Validación**

1. El emisor del E43 es quien realiza el gasto
    
2. El monto total debe estar dentro de los límites establecidos por DGII
    
3. Debe documentar claramente el concepto del gasto
    
4. Generalmente todo el monto es exento de ITBIS
    
5. El "comprador" en el JSON representa al proveedor/vendedor
    
6. **El campo fechavencimientosecuencia es obligatorio para E43**
    

#### **Ejemplo Práctico E43**

``` json
{
  "encabezado": {
    "version": "1.0",
    "idDoc": {
      "tipoeCF": "43",
      "encf": "E430000000001",
      "indicadorMontoGravado": 0,
      "tipoIngresos": "01",
      "tipoPago": 1,
      "terminoPago": "CONTADO",
      "fechavencimientosecuencia": "31-12-2025",
      "tablaFormasPago": {
        "formaDePago": [
          {
            "formaPago": "1",
            "montoPago": "2500.00"
          }
        ]
      }
    },
    "emisor": {
      "rncEmisor": "101742186",
      "razonSocialEmisor": "Empresa Compradora SRL",
      "nombreComercial": "Empresa Compradora",
      "direccionEmisor": "Calle Principal No. 123, Santo Domingo",
      "numeroFacturaInterna": "GM-2025-001",
      "zonaVenta": "GASTOS-MENORES",
      "fechaEmision": "15-05-2025"
    },
    "comprador": {
      "RNCComprador": "",
      "razonSocialComprador": "María Rodríguez - Vendedora Informal",
      "direccionComprador": "Mercado Central, Puesto 45, Santo Domingo",
      "codigoInternoComprador": "PROV-GM-001"
    },
    "totales": {
      "MontoExento": "2500.00",
      "MontoDescuento": null,
      "MontoRecargo": null,
      "MontoTotal": "2500.00"
    }
  },
  "detallesItems": {
    "items": [
      {
        "numeroLinea": 1,
        "indicadorFacturacion": 4,
        "nombreItem": "Compra de materiales de oficina - útiles varios",
        "indicadorBienoServicio": 1,
        "cantidadItem": "1.00",
        "unidadMedida": 31,
        "precioUnitarioItem": "2500.00",
        "montoItem": "2500.00",
        "tablaImpuestoAdicional": {}
      }
    ]
  },
  "Subtotales": {},
  "descuentosORecargos": {
    "descuentoORecargo": []
  },
  "paginacion": {},
  "informacionReferencia": {}
}

 ```

#### **Casos de Uso Comunes E43**

- Compras en mercados locales
    
- Gastos de transporte menor (taxis, estacionamiento)
    
- Materiales de oficina a vendedores informales
    
- Servicios de mensajería local
    
- Alimentación en establecimientos sin facturación
    
- Pequeñas reparaciones y mantenimiento
    

---

### E44: Regímenes Especiales

#### **Propósito**

El Comprobante de Regímenes Especiales (E44) se utiliza para transacciones con contribuyentes acogidos a regímenes tributarios especiales, como zonas francas, turismo, energía renovable, y otros sectores con tratamiento fiscal diferenciado.

#### **Características Principales**

- Para transacciones con contribuyentes en regímenes especiales
    
- Tratamiento fiscal diferenciado según el régimen
    
- Puede incluir exenciones totales o parciales
    
- Requiere identificación del tipo de régimen
    
- Documentación especial según normativa del régimen
    

#### **Campos Obligatorios Específicos**

- `encabezado.idDoc.tipoeCF`: "44"
    
- `encabezado.comprador.RNCComprador`: Obligatorio
    
- Indicadores especiales según el régimen aplicable
    
- Documentación de exenciones o beneficios fiscales
    

#### **Reglas de Validación**

1. El comprador debe estar registrado en un régimen especial
    
2. Aplicar correctamente las exenciones según el régimen
    
3. Documentar el tipo de régimen especial
    
4. Cumplir requisitos específicos de cada régimen
    
5. Mantener documentación de respaldo del régimen
    
6. **El campo fechavencimientosecuencia es obligatorio para E44**
    

#### **Ejemplo Práctico E44 - Zona Franca**

``` json
{
  "encabezado": {
    "version": "1.0",
    "idDoc": {
      "tipoeCF": "44",
      "encf": "E440000000001",
      "indicadorMontoGravado": 0,
      "tipoIngresos": "01",
      "tipoPago": 2,
      "fechaLimitePago": "15-06-2025",
      "terminoPago": "30 DIAS",
      "fechavencimientosecuencia": "31-12-2025",
      "tablaFormasPago": {
        "formaDePago": [
          {
            "formaPago": "2",
            "montoPago": "250000.00"
          }
        ]
      }
    },
    "emisor": {
      "rncEmisor": "101742186",
      "razonSocialEmisor": "Proveedor Nacional SRL",
      "nombreComercial": "Proveedor Nacional",
      "direccionEmisor": "Calle Industrial No. 789, Santo Domingo",
      "numeroFacturaInterna": "ZF-2025-001",
      "zonaVenta": "ZONA-FRANCA",
      "fechaEmision": "15-05-2025"
    },
    "comprador": {
      "RNCComprador": "130123456",
      "razonSocialComprador": "Empresa Zona Franca SA",
      "correoComprador": "compras@zonafranca.com",
      "direccionComprador": "Parque Industrial Zona Franca, Santiago",
      "codigoInternoComprador": "ZF-CLI-001"
    },
    "totales": {
      "MontoExento": "250000.00",
      "MontoImpuestoAdicional": "0.00",
      "MontoTotal": "250000.00"
    }
  },
  "detallesItems": {
    "items": [
      {
        "numeroLinea": 1,
        "indicadorFacturacion": 4,
        "nombreItem": "Materia prima para manufactura - Componentes electrónicos",
        "indicadorBienoServicio": 1,
        "cantidadItem": "1000.00",
        "unidadMedida": 43,
        "precioUnitarioItem": "250.00",
        "montoItem": "250000.00"
      }
    ]
  },
  "Subtotales": {},
  "descuentosORecargos": {
    "descuentoORecargo": []
  },
  "paginacion": {},
  "informacionReferencia": {}
}

 ```

#### **Tipos de Regímenes Especiales**

- Zonas Francas
    
- Sector Turismo
    
- Energías Renovables
    
- Sector Agropecuario
    
- Frontera
    
- Cinematografía
    
- Otros regímenes autorizados por ley
    

---

### E45: Gubernamental

#### **Propósito**

El Comprobante Gubernamental (E45) se utiliza para transacciones con entidades del Estado dominicano, incluyendo ministerios, ayuntamientos, instituciones autónomas y descentralizadas.

#### **Características Principales**

- Exclusivo para ventas al Estado
    
- El comprador debe ser una entidad gubernamental
    
- Puede incluir retenciones especiales
    
- Requiere información específica de la entidad estatal
    
- Proceso de pago según normativa gubernamental
    

#### **Campos Obligatorios Específicos**

- `encabezado.idDoc.tipoeCF`: "45"
    
- `encabezado.comprador.RNCComprador`: RNC de la entidad gubernamental
    
- `encabezado.comprador.razonSocialComprador`: Nombre oficial de la entidad
    
- Información de orden de compra gubernamental (si aplica)
    

#### **Reglas de Validación**

1. El RNC del comprador debe corresponder a una entidad estatal
    
2. Cumplir con requisitos de contratación pública
    
3. Incluir referencias de órdenes de compra o contratos
    
4. Aplicar retenciones según normativa
    
5. Documentación adicional según tipo de entidad
    
6. **El campo fechavencimientosecuencia es obligatorio para E45**
    

#### **Ejemplo Práctico E45**

``` json
{
  "encabezado": {
    "version": "1.0",
    "idDoc": {
      "tipoeCF": "45",
      "encf": "E450000000001",
      "indicadorMontoGravado": 1,
      "tipoIngresos": "01",
      "tipoPago": 2,
      "fechaLimitePago": "30-06-2025",
      "terminoPago": "60 DIAS",
      "fechavencimientosecuencia": "31-12-2025",
      "tablaFormasPago": {
        "formaDePago": [
          {
            "formaPago": "2",
            "montoPago": "590000.00"
          }
        ]
      }
    },
    "emisor": {
      "rncEmisor": "101742186",
      "razonSocialEmisor": "Proveedor Servicios Gobierno SRL",
      "nombreComercial": "Proveedor Gobierno",
      "direccionEmisor": "Av. Principal No. 456, Santo Domingo",
      "numeroFacturaInterna": "GOB-2025-001",
      "zonaVenta": "GOBIERNO",
      "fechaEmision": "15-05-2025"
    },
    "comprador": {
      "RNCComprador": "401000013",
      "razonSocialComprador": "Ministerio de Educación de la República Dominicana",
      "correoComprador": "compras@minerd.gob.do",
      "direccionComprador": "Av. Máximo Gómez, Santo Domingo",
      "numeroOrdenCompra": "OC-MINERD-2025-12345",
      "fechaOrdenCompra": "01-05-2025",
      "codigoInternoComprador": "MINERD"
    },
    "totales": {
      "MontoGravadoTotal": "500000.00",
      "MontoGravadoI1": "500000.00",
      "MontoGravadoI2": "0.00",
      "MontoGravadoI3": "0.00",
      "MontoExento": "0.00",
      "ITBIS1": 18,
      "TotalITBIS": "90000.00",
      "TotalITBIS1": "90000.00",
      "TotalITBIS2": "0.00",
      "TotalITBIS3": "0.00",
      "MontoImpuestoAdicional": "0.00",
      "MontoTotal": "590000.00"
    }
  },
  "detallesItems": {
    "items": [
      {
        "numeroLinea": 1,
        "indicadorFacturacion": 1,
        "nombreItem": "Servicio de mantenimiento de equipos informáticos - Contrato anual",
        "indicadorBienoServicio": 2,
        "cantidadItem": "1.00",
        "unidadMedida": 43,
        "precioUnitarioItem": "500000.00",
        "montoItem": "500000.00"
      }
    ]
  },
  "Subtotales": {},
  "descuentosORecargos": {
    "descuentoORecargo": []
  },
  "paginacion": {},
  "informacionReferencia": {}
}

 ```

#### **Entidades Gubernamentales Comunes**

- Ministerios
    
- Ayuntamientos
    
- Instituciones Autónomas
    
- Empresas Públicas
    
- Instituciones Descentralizadas
    
- Poder Judicial
    
- Poder Legislativo
    

---

### E46: Exportaciones

#### **Propósito**

El Comprobante de Exportación (E46) documenta las ventas de bienes o servicios a clientes en el extranjero. Estas operaciones están exentas de ITBIS según la legislación dominicana.

#### **Características Principales**

- Para ventas al exterior
    
- ITBIS a tasa 0 % (`indicadorFacturacion` = 3)
    
- Requiere identificación del comprador extranjero
    
- Debe incluir información de exportación
    
- Generalmente en moneda extranjera
    

#### **Campos Obligatorios Específicos**

- `encabezado.idDoc.tipoeCF`: "46"
    
- `encabezado.comprador.identificadorextranjero`: Obligatorio
    
- `encabezado.comprador.razonSocialComprador`: Nombre del comprador extranjero
    
- `encabezado.OtraMoneda`: Generalmente requerido
    
- Información de embarque/exportación
    

#### **Reglas de Validación**

1. El comprador debe ser extranjero (no RNC dominicano)
    
2. Todo el monto va a tasa 0 %, en `MontoGravadoI3` (el E46 no admite `MontoExento`)
    
3. Debe incluir información de moneda extranjera
    
4. Documentar el país de destino
    
5. Cumplir con requisitos aduaneros
    
6. **El campo fechavencimientosecuencia es obligatorio para E46**
    

#### **Ejemplo Práctico E46**

``` json
{
  "encabezado": {
    "version": "1.0",
    "idDoc": {
      "tipoeCF": "46",
      "encf": "E460000000001",
      "indicadorMontoGravado": 0,
      "tipoIngresos": "01",
      "tipoPago": 2,
      "fechaLimitePago": "30-06-2025",
      "terminoPago": "45 DIAS",
      "fechavencimientosecuencia": "31-12-2025",
      "tablaFormasPago": {
        "formaDePago": [
          {
            "formaPago": "3",
            "montoPago": "2950000.00"
          }
        ]
      }
    },
    "emisor": {
      "rncEmisor": "101742186",
      "razonSocialEmisor": "Exportadora Dominicana SRL",
      "nombreComercial": "Exportadora Dominicana",
      "direccionEmisor": "Zona Industrial Herrera, Santo Domingo",
      "numeroFacturaInterna": "EXP-2025-001",
      "zonaVenta": "EXPORTACION",
      "fechaEmision": "15-05-2025"
    },
    "comprador": {
      "RNCComprador": "",
      "identificadorextranjero": "TAX-US-123456789",
      "razonSocialComprador": "International Trading Company LLC",
      "correoComprador": "purchasing@intltradingco.com",
      "direccionComprador": "1234 Commerce Street, Miami, FL 33101, USA",
      "fechaEntrega": "30-05-2025",
      "numeroOrdenCompra": "PO-USA-2025-5678",
      "fechaOrdenCompra": "01-05-2025",
      "codigoInternoComprador": "USA-CLI-001"
    },
    "totales": {
      "MontoGravadoTotal": "2950000.00",
      "MontoGravadoI3": "2950000.00",
      "ITBIS3": 0,
      "TotalITBIS": "0.00",
      "TotalITBIS3": "0.00",
      "MontoTotal": "2950000.00"
    },
    "OtraMoneda": {
      "TipoMoneda": "USD",
      "TipoCambio": "59.00",
      "MontoGravado3OtraMoneda": "50000.00",
      "MontoGravadoTotalOtraMoneda": "50000.00",
      "TotalITBISOtraMoneda": "0.00",
      "TotalITBIS3OtraMoneda": "0.00",
      "MontoTotalOtraMoneda": "50000.00"
    }
  },
  "detallesItems": {
    "items": [
      {
        "numeroLinea": 1,
        "indicadorFacturacion": 3,
        "nombreItem": "Productos manufacturados - Componentes electrónicos",
        "indicadorBienoServicio": 1,
        "cantidadItem": "5000.00",
        "unidadMedida": 43,
        "precioUnitarioItem": "590.00",
        "montoItem": "2950000.00",
        "OtraMonedaDetalle": {
          "PrecioOtraMoneda": "10.00",
          "MontoItemOtraMoneda": "50000.00"
        }
      }
    ]
  },
  "Subtotales": {},
  "descuentosORecargos": {
    "descuentoORecargo": []
  },
  "paginacion": {},
  "informacionReferencia": {}
}

 ```

#### **Casos de Uso Comunes E46**

- Exportación de productos manufacturados
    
- Servicios prestados a clientes extranjeros
    
- Venta de productos agrícolas al exterior
    
- Servicios digitales a clientes internacionales
    
- Exportación de artesanías y productos locales
    

---

### E47: Comprobante de Pagos al Exterior de Servicios

#### **Propósito**

El Comprobante de Pagos al Exterior de Servicios (E47) se utiliza para documentar pagos realizados a proveedores extranjeros por servicios recibidos. Este tipo de comprobante es importante para la deducibilidad fiscal y el cumplimiento de las obligaciones de retención en la fuente cuando apliquen.

#### **Características Principales**

- Para pagos de servicios a proveedores en el exterior
    
- El proveedor no tiene presencia fiscal en República Dominicana
    
- Generalmente exento de ITBIS (servicios prestados desde el exterior)
    
- Requiere identificación del proveedor extranjero
    
- Puede incluir retenciones en la fuente
    
- Transacción típicamente en moneda extranjera
    

#### **Campos Obligatorios Específicos**

- `encabezado.idDoc.tipoeCF`: "47"
    
- `encabezado.idDoc.fechavencimientosecuencia`: Obligatorio
    
- `encabezado.emisor`: Empresa dominicana que realiza el pago
    
- `encabezado.comprador`: Proveedor extranjero del servicio
    
    - `identificadorextranjero`: Obligatorio (Tax ID, VAT, etc.)
        
    - `razonSocialComprador`: Nombre del proveedor extranjero
        
    - `direccionComprador`: Dirección en el país de origen
        
- `encabezado.OtraMoneda`: Generalmente requerido para moneda extranjera
    

#### **Reglas de Validación**

1. El proveedor debe ser extranjero (usar identificadorextranjero, no RNC)
    
2. Los servicios deben ser prestados desde el exterior
    
3. Generalmente todo el monto es exento de ITBIS
    
4. Debe incluir información completa del proveedor extranjero
    
5. Si aplica retención en la fuente, documentarla apropiadamente
    
6. Incluir conversión de moneda cuando el pago sea en moneda extranjera
    
7. **El campo fechavencimientosecuencia es obligatorio para E47**
    

#### **Ejemplo Práctico E47**

``` json
{
  "encabezado": {
    "version": "1.0",
    "idDoc": {
      "tipoeCF": "47",
      "encf": "E470000000001",
      "indicadorMontoGravado": 0,
      "tipoIngresos": "01",
      "tipoPago": 2,
      "fechaLimitePago": "15-06-2025",
      "terminoPago": "30 DIAS",
      "fechavencimientosecuencia": "31-12-2025",
      "tablaFormasPago": {
        "formaDePago": [
          {
            "formaPago": "2",
            "montoPago": "1475000.00"
          }
        ]
      }
    },
    "emisor": {
      "rncEmisor": "101742186",
      "razonSocialEmisor": "Empresa Tecnología Dominicana SRL",
      "nombreComercial": "Empresa Tecnología",
      "direccionEmisor": "Calle Tecnológica No. 789, Santo Domingo",
      "numeroFacturaInterna": "PSE-2025-001",
      "zonaVenta": "SERVICIOS-EXTERIOR",
      "fechaEmision": "15-05-2025"
    },
    "comprador": {
      "RNCComprador": "",
      "identificadorextranjero": "EIN-98-7654321",
      "razonSocialComprador": "Global Software Solutions Inc.",
      "correoComprador": "billing@globalsoftware.com",
      "direccionComprador": "500 Tech Park Drive, San Francisco, CA 94105, USA",
      "codigoInternoComprador": "PROV-USA-001"
    },
    "totales": {
      "MontoExento": "1475000.00",
      "MontoDescuento": null,
      "MontoRecargo": null,
      "MontoTotal": "1475000.00"
    },
    "OtraMoneda": {
      "TipoMoneda": "USD",
      "TipoCambio": "59.00",
      "MontoExentoOtraMoneda": "25000.00",
      "MontoTotalOtraMoneda": "25000.00"
    }
  },
  "detallesItems": {
    "items": [
      {
        "numeroLinea": 1,
        "indicadorFacturacion": 4,
        "nombreItem": "Licencias de software empresarial - Suscripción anual",
        "indicadorBienoServicio": 2,
        "cantidadItem": "1.00",
        "unidadMedida": 43,
        "precioUnitarioItem": "1475000.00",
        "montoItem": "1475000.00",
        "tablaImpuestoAdicional": {},
        "OtraMonedaDetalle": {
          "PrecioOtraMoneda": "25000.00",
          "MontoItemOtraMoneda": "25000.00"
        }
      }
    ]
  },
  "Subtotales": {},
  "descuentosORecargos": {
    "descuentoORecargo": []
  },
  "paginacion": {},
  "informacionReferencia": {}
}

 ```

#### **Casos de Uso Comunes E47**

- Licencias de software internacional
    
- Servicios de consultoría extranjera
    
- Servicios de hosting y cloud computing
    
- Publicidad en plataformas internacionales
    
- Servicios profesionales de firmas extranjeras
    
- Suscripciones a plataformas SaaS internacionales
    
- Servicios de capacitación y entrenamiento en línea
    

---

## Catálogos y Códigos

### Catálogo de Formas de Pago

Son los códigos de la DGII para `formaPago` (tabla de formas de pago del e-CF):

| Código | Descripción |
| --- | --- |
| 1 | Efectivo |
| 2 | Cheque / Transferencia / Depósito |
| 3 | Tarjeta de Débito / Crédito |
| 4 | Venta a Crédito |
| 5 | Bonos o Certificados de Regalo |
| 6 | Permuta |
| 7 | Nota de Crédito |
| 8 | Otras Formas de Pago |

> **Corrección (octubre 2026):** versiones anteriores de esta tabla tenían códigos que no son los de la DGII. Si su integración la usaba, cambie el mapeo:
>
> | Forma de pago | Código viejo | Se declaraba como | Código correcto |
> | --- | --- | --- | --- |
> | Transferencia | 3 | Tarjeta | **2** |
> | Tarjeta de crédito | 4 | Venta a crédito | **3** |
> | Tarjeta de débito | 5 | Bonos o certificados | **3** |
> | Bonos o certificados | 8 | Otras formas de pago | **5** |
> | Otras formas de pago | 9 | No existe: la DGII no lo admite | **8** |
>
> Efectivo, cheque, permuta y nota de crédito no cambiaron.

### Catálogo de Unidades de Medida

Son los códigos de la DGII para `unidadMedida` (tabla del formato e-CF). El campo es opcional: si no aplica, no lo envíe. Unidad es `43`; `1` es barril.

| Código | Abreviatura | Descripción |
| --- | --- | --- |
| 1 | BARR | Barril |
| 2 | BOL | Bolsa |
| 3 | BOT | Bote |
| 4 | BULTO | Bultos |
| 5 | BOTELLA | Botella |
| 6 | CAJ | Caja/Cajón |
| 7 | CAJETILLA | Cajetilla |
| 8 | CM | Centímetro |
| 9 | CIL | Cilindro |
| 10 | CONJ | Conjunto |
| 11 | CONT | Contenedor |
| 12 | DÍA | Día |
| 13 | DOC | Docena |
| 14 | FARD | Fardo |
| 15 | GL | Galones |
| 16 | GRAD | Grado |
| 17 | GR | Gramo |
| 18 | GRAN | Granel |
| 19 | HOR | Hora |
| 20 | HUAC | Huacal |
| 21 | KG | Kilogramo |
| 22 | kWh | Kilovatio Hora |
| 23 | LB | Libra |
| 24 | LITRO | Litro |
| 25 | LOT | Lote |
| 26 | M | Metro |
| 27 | M2 | Metro Cuadrado |
| 28 | M3 | Metro Cúbico |
| 29 | MMBTU | Millones de Unidades Térmicas |
| 30 | MIN | Minuto |
| 31 | PAQ | Paquete |
| 32 | PAR | Par |
| 33 | PIE | Pie |
| 34 | PZA | Pieza |
| 35 | ROL | Rollo |
| 36 | SOBR | Sobre |
| 37 | SEG | Segundo |
| 38 | TANQUE | Tanque |
| 39 | TONE | Tonelada |
| 40 | TUB | Tubo |
| 41 | YD | Yarda |
| 42 | YD2 | Yarda cuadrada |
| 43 | UND | Unidad |
| 44 | EA | Elemento |
| 45 | MILLAR | Millar |
| 46 | SAC | Saco |
| 47 | LAT | Lata |
| 48 | DIS | Display |
| 49 | BID | Bidón |
| 50 | RAC | Ración |
| 51 | Q | Quintal |
| 52 | GRT | Toneladas de registro bruto |
| 53 | P2 | Pie Cuadrado |
| 54 | PAX | Pasajero |
| 55 | PULG | Pulgadas |
| 56 | STAY | Parqueo Barcos En Muelle |
| 57 | BDJ | Bandeja |

> **Corrección (noviembre 2026):** la tabla anterior no era la de la DGII. Si su integración la usaba, cambie el mapeo:
>
> | Unidad | Código viejo | Código correcto |
> | --- | --- | --- |
> | Unidad | 1 | **43** |
> | Caja | 2 | **6** |
> | Kilogramo | 3 | **21** |
> | Gramo | 4 | **17** |
> | Libra | 5 | **23** |
> | Metro | 7 | **26** |
> | Centímetro | 8 | **8** |
> | Pulgada | 9 | **55** |
> | Pie | 10 | **33** |
> | Yarda | 11 | **41** |
> | Litro | 12 | **24** |
> | Galón | 14 | **15** |
> | Docena | 15 | **13** |
> | Paquete | 16 | **31** |
> | Hora | 18 | **19** |
> | Día | 19 | **12** |
>
> Onza (6), mililitro (13), servicio (17) y mes (20) no existen en la tabla de la DGII. Para esas líneas use otra unidad o no envíe el campo.

### Catálogo de Tipos de Moneda

| Código | Descripción |
| --- | --- |
| USD | Dólar Estadounidense |
| EUR | Euro |
| CAD | Dólar Canadiense |
| GBP | Libra Esterlina |
| CHF | Franco Suizo |
| JPY | Yen Japonés |
| DOP | Peso Dominicano |

### Catálogo de Indicadores de Facturación

Son los códigos de la DGII para `indicadorFacturacion`: dicen cómo se grava la línea, no si es un producto o un servicio (eso lo dice `indicadorBienoServicio`).

| Código | Descripción | Total donde se suma la línea |
| --- | --- | --- |
| 0 | No facturable | Ninguno |
| 1 | ITBIS 18 % | `MontoGravadoI1` |
| 2 | ITBIS 16 % | `MontoGravadoI2` |
| 3 | ITBIS 0 % | `MontoGravadoI3` |
| 4 | Exento | `MontoExento` |

El indicador tiene que cuadrar con los totales: el API exige `ITBIS1`, `ITBIS2`, `ITBIS3` o `MontoExento` según el indicador de cada línea. No todos los tipos admiten todos los totales: E43 y E44 solo llevan `MontoExento`, y E46 solo `MontoGravadoI3`.

---

## Códigos de Respuesta del Sistema

### Códigos de Estado

Son los códigos de estado de la DGII:

| Código | Estado | Descripción |
| --- | --- | --- |
| 0 | No encontrado | La DGII no tiene registro del eCF consultado |
| 1 | Aceptado | El eCF fue aceptado sin observaciones |
| 2 | Rechazado | El eCF fue rechazado; el motivo viene en `mensajes` |
| 3 | En Proceso | La DGII todavía lo está validando |
| 4 | Aceptado Condicional | El eCF fue aceptado con observaciones (vienen en `mensajes`) |

#### Facturas de Consumo (E32) menores a RD$250,000

Estas facturas se envían a la DGII como **Resumen de Factura de Consumo (RFCE)** y la respuesta trae el **estado real de la DGII**: `Aceptado`, `Aceptado Condicional` o `Rechazado`, con su `codigo`, `mensajes` y `secuenciaUtilizada`. No llevan `trackId`.

- Ya no se devuelve el estado `Enviado Resumen`: si ve `Rechazado`, el motivo está en `mensajes`, igual que en el QR de la DGII.
- Si el resumen no llega a la DGII (problema de conexión o de autenticación), la respuesta es **HTTP 502** con `estado: "Error de envío"` y el motivo en `mensaje`. En ese caso la DGII no recibió nada: reintente el envío con el mismo eNCF.

### Códigos de Error Comunes

El motivo exacto de un rechazo viene en `mensajes[].valor`, y su número en `mensajes[].codigo` (no confundir con `codigo`, que es el estado). Guíese por ese texto.

| Código | Descripción | Solución |
| --- | --- | --- |
| 1001 | RNC del emisor inválido | Verificar que el RNC esté correctamente registrado |
| 1209 | «Este número de secuencia ya ha sido utilizado» | El número quedó gastado: corrija y emita con un e-NCF nuevo |
| 1003 | Formato de fecha inválido | Usar formato DD-MM-YYYY |
| 1004 | Monto total no coincide | Verificar cálculos de totales |
| 1005 | ITBIS calculado incorrectamente | Recalcular ITBIS según tasas vigentes |
| 1934 | Campo MontoGravadoI1 no coincide | Verificar que la suma de ítems gravados coincida |
| 1964 | Campo MontoExento no coincide | Verificar que la suma de ítems exentos coincida |
| 2001 | RNC del comprador inválido | Verificar RNC del comprador en DGII |
| 2002 | Identificador extranjero requerido | Incluir identificador para compradores extranjeros |
| 3001 | Referencia a eCF inexistente | Verificar que el eCF referenciado exista (NC/ND) |
| 3002 | Monto de NC excede original | El monto de la nota de crédito no puede ser mayor al original |

---

## Tablas de Comparación

### Comparación de Tipos de eCF

| Característica | E31 | E32 | E33 | E34 | E41 | E43 | E44 | E45 | E46 | E47 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **RNC Comprador Obligatorio** | Sí | No | Según original | Según original | Opcional | Opcional | Sí | Sí | No | No |
| **Crédito Fiscal ITBIS** | Sí | No | Según original | Según original | No | No | Según régimen | Sí | No | No |
| **Permite Exento** | Sí | Sí | Sí | Sí | Sí | Sí | Sí | Sí | No (tasa 0 %) | Sí (todo) |
| **Requiere Referencia** | No | No | Sí | Sí | No | No | No | No | No | No |
| **Moneda Extranjera** | Opcional | Opcional | Opcional | Opcional | Opcional | Opcional | Opcional | Opcional | Común | Común |
| **Emisor** | Vendedor | Vendedor | Vendedor | Vendedor | Comprador | Comprador | Vendedor | Vendedor | Vendedor | Comprador |
| **fechavencimientosecuencia** | Req | No | Req | No | Req | Req | Req | Req | Req | Req |

### Comparación de Campos por Tipo

| Campo | E31 | E32 | E33 | E34 | E41 | E43 | E44 | E45 | E46 | E47 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| RNCComprador | ✓ Req | ○ Opc | ✓ Según | ✓ Según | ○ Opc | ○ Opc | ✓ Req | ✓ Req | ✗ No | ✗ No |
| identificadorextranjero | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ✗ No | ✓ Req | ✓ Req |
| MontoGravado | ✓ Req | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ✓ Solo I3 | ○ Opc |
| MontoExento | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ✓ Común | ✓ Común | ✓ Común | ○ Opc | ✗ No | ✓ Común |
| informacionReferencia | ✗ No | ✗ No | ✓ Req | ✓ Req | ✗ No | ✗ No | ✗ No | ✗ No | ✗ No | ✗ No |
| OtraMoneda | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ○ Opc | ✓ Común | ✓ Común |
| fechavencimientosecuencia | ✓ Req | ✗ No | ✓ Req | ✗ No | ✓ Req | ✓ Req | ✓ Req | ✓ Req | ✓ Req | ✓ Req |

**Leyenda:**

- ✓ Req: Requerido
    
- ○ Opc: Opcional
    
- ✗ No: No aplica
    
- ✓ Según: Según documento original
    
- ✓ Común: Comúnmente usado
    

---

## Mejores Prácticas

### 1\. Validación de Datos Antes del Envío

**Validaciones Críticas:**

- Verificar que el RNC del emisor esté activo en DGII
    
- Validar formato y existencia del RNC del comprador (cuando aplique)
    
- Confirmar que la secuencia del eCF no haya sido utilizada
    
- Verificar cálculos matemáticos de totales e ITBIS
    
- Validar formato de fechas (DD-MM-YYYY)
    
- Confirmar que los montos en otra moneda coincidan con el tipo de cambio
    
- **Verificar que fechavencimientosecuencia esté presente para todos los tipos excepto E32 y E34**
    
- Validar que fechavencimientosecuencia no se incluya en E32 y E34
    

**Ejemplo de Validación de Totales:**

``` javascript
// Validar que MontoTotal sea correcto
const montoCalculado = 
  parseFloat(MontoGravadoTotal) + 
  parseFloat(MontoExento) + 
  parseFloat(TotalITBIS) + 
  parseFloat(MontoImpuestoAdicional) - 
  parseFloat(MontoDescuento || 0) + 
  parseFloat(MontoRecargo || 0);
if (Math.abs(montoCalculado - MontoTotal) > 0.01) {
  console.error("Error: MontoTotal no coincide con el cálculo");
}

 ```

### 2\. Manejo de Errores y Reintentos

**La regla:** reintentar es seguro con **el mismo e-NCF**. Un número nuevo solo es correcto cuando la DGII ya rechazó el comprobante y el número quedó gastado. En cualquier otro caso, un número nuevo puede crear una segunda factura fiscal por la misma venta.

**Qué hacer según la respuesta:**

| Respuesta | Qué significa | Qué hacer | e-NCF |
| --- | --- | --- | --- |
| Sin respuesta, tiempo agotado, error de red o HTTP 5xx | No se sabe si llegó | Reenviar el **mismo JSON** | El mismo |
| HTTP 502, `estado: "Error de envío"` (E32 menor a RD$250,000) | La DGII no recibió nada | Reenviar el mismo JSON | El mismo |
| HTTP 202, `estado: "Pendiente DGII"` | Quedó firmado y guardado, y DOSA lo enviará solo cuando la DGII responda | Imprimir con ese QR y la `leyendaContingencia`. No reemplazarlo. | El mismo |
| HTTP 409, `error: "e-NCF pendiente de la DGII con otro contenido"` | Ese e-NCF ya se firmó con otros datos | No se puede cambiar. Esperar el resultado de la DGII. | El mismo |
| HTTP 200, `Aceptado` o `Aceptado Condicional` | Es válido | Guardarlo y no volver a emitirlo. Para corregirlo, nota de crédito (E34). | No reemitir |
| HTTP 200, `Rechazado` con `secuenciaUtilizada: false` | No es válido y el número sigue libre | Corregir y reenviar | El mismo |
| HTTP 200, `Rechazado` con `secuenciaUtilizada: true` o sin ese campo | No es válido y el número quedó gastado | Corregir y emitir de nuevo | **Uno nuevo** |
| `Rechazado` con el mensaje 1209 | Se reutilizó un número gastado | Corregir y emitir de nuevo | **Uno nuevo** |
| HTTP 4xx de validación (por ejemplo, un campo obligatorio) | No se aceptó el envío | Corregir y reenviar | El mismo |

Ante la duda con un **rechazo**, use un número nuevo: el comprobante rechazado no tiene validez fiscal, así que lo único que se pierde es un número. Ante la duda con **cualquier otra respuesta**, use el mismo.

**Cómo evitar una doble emisión:**

- **Guarde el e-NCF antes de enviar.** Asigne el número al documento y guarde esa relación antes del primer envío, para que cada reintento use el mismo.
- **El e-NCF es la llave.** El API no tiene una llave de idempotencia aparte: identifica el comprobante por e-NCF, RNC del emisor y ambiente. Reenviar un e-NCF aceptado devuelve el mismo comprobante sin firmarlo ni enviarlo otra vez; reenviar uno pendiente devuelve el mismo QR.
- **Para saber cómo quedó, reenvíe el mismo JSON.** Si está aceptado o pendiente, la respuesta lo dice y no se emite nada nuevo. Si había quedado rechazado, se firma y se envía de nuevo.
- **No cambie los datos de un e-NCF aceptado.** El API no compara el contenido: si reenvía otros datos con ese número, responde HTTP 200 con el comprobante original, sin avisar.
- **Un solo envío a la vez por documento.** No mande el mismo e-NCF en paralelo; use un bloqueo por documento.

**Reintentos automáticos:**

- Solo cuando no hubo respuesta o fue un HTTP 5xx: hasta 3 intentos, con espera creciente (1 s, 3 s, 5 s)
    
- Si siguen fallando, deje el documento pendiente y vuelva a intentar más tarde con el mismo e-NCF
    
- Guardar log de todos los intentos y respuestas
    

**Manejo de Respuestas:**

``` javascript
// Decida por `estado`: `codigo` es lo mismo en número y viene como texto.
switch (response.estado) {
  case "Aceptado":
    guardarComprobante(response);
    break;
  case "Aceptado Condicional":
    // Es válido, pero con observaciones que hay que revisar
    revisarMensajes(response.mensajes);
    guardarComprobante(response);
    break;
  case "Rechazado":
    // Corregir según `mensajes`. Número nuevo salvo que secuenciaUtilizada sea false.
    procesarRechazo(response.mensajes, response.secuenciaUtilizada !== false);
    break;
  case "Pendiente DGII": // HTTP 202
  case "En Proceso":
    // Imprimir con este QR y la leyenda si viene. No emitir con otro número.
    guardarPendiente(response);
    break;
}

 ```

### 3\. Gestión de Secuencias

**Recomendaciones:**

- Mantener un contador secuencial por tipo de eCF
    
- Una secuencia es de un solo documento: guardar la relación antes de enviar. Cuándo reintentar con la misma y cuándo usar otra está en [Manejo de Errores y Reintentos](#2-manejo-de-errores-y-reintentos)
    
- Implementar bloqueos para evitar duplicados en ambientes concurrentes
    
- Guardar registro de todas las secuencias utilizadas
    
- Implementar alertas cuando se acerque al límite de secuencias
    

**Formato de Secuencia:**

```
E + tipo (2 dígitos) + secuencia (10 dígitos) = 13 caracteres
Ejemplo: E310000000001, E320000000001

 ```

### 4\. Seguridad y Autenticación

**Protección de API Keys:**

- Nunca incluir API Keys en código fuente
    
- Usar variables de entorno o servicios de gestión de secretos
    
- Rotar API Keys periódicamente
    
- Implementar diferentes keys para ambientes (desarrollo, producción)
    
- Monitorear uso de API Keys para detectar anomalías
    

**Ejemplo de Configuración Segura:**

``` javascript
// Usar variables de entorno
const apiKey = process.env.DGII_API_KEY;
// Nunca hacer esto:
// const apiKey = "821b20c1-289e-4320-8def-a2b5f1dd2c39";

 ```

### 5\. Logging y Auditoría

**Información a Registrar:**

- Timestamp de cada solicitud
    
- Datos completos del request (sin información sensible)
    
- Respuesta completa del servidor
    
- TrackId asignado por DGII
    
- Código de seguridad generado
    
- Errores y advertencias
    
- Usuario que generó el eCF
    

**Estructura de Log Recomendada:**

``` json
{
  "timestamp": "2025-05-15T14:46:41Z",
  "tipo_ecf": "32",
  "encf": "E320000000004",
  "track_id": "4cdc9917-8008-4564-a85c-c3f13a4e0ec8",
  "codigo_seguridad": "EAkbyR",
  "estado": "Aceptado Condicional",
  "monto_total": 3491052.52,
  "comprador": "FRANÇOIS XAVIER DE LA POYPE",
  "mensajes": [
    {
      "codigo": 1934,
      "descripcion": "El campo MontoGravadoI1..."
    }
  ]
}

 ```

### 6\. Manejo de Moneda Extranjera

**Consideraciones:**

- Usar tipo de cambio oficial del Banco Central
    
- Mantener consistencia entre montos en DOP y moneda extranjera
    
- Redondear correctamente (2 decimales para DOP, según moneda para extranjera)
    
- Documentar la fuente del tipo de cambio utilizado
    
- Actualizar tipos de cambio diariamente
    

**Cálculo de Conversión:**

``` javascript
// Ejemplo de conversión USD a DOP
const montoDOP = montoUSD * tipoCambio;
const montoDOPRedondeado = Math.round(montoDOP * 100) / 100;
// Validar que la conversión sea consistente
const diferenciaPermitida = 1.00; // 1 peso de diferencia máxima
if (Math.abs(montoDOPCalculado - montoDOPDeclarado) > diferenciaPermitida) {
  console.warn("Advertencia: Diferencia en conversión de moneda");
}

 ```

### 7\. Testing y Ambiente de Pruebas

**Estrategia de Testing:**

- Usar ambiente de pruebas de DGII antes de producción
    
- Crear casos de prueba para cada tipo de eCF
    
- Probar escenarios de error comunes
    
- Validar cálculos con diferentes combinaciones de tasas
    
- Probar con diferentes formas de pago
    

**URLs de Ambientes:**

```
Pruebas: https://ecf.dgii.gov.do/testecf/...
Producción: https://ecf.dgii.gov.do/ecf/...

 ```

### 8\. Optimización de Performance

**Recomendaciones:**

- Implementar caché para datos estáticos (catálogos, tasas)
    
- Procesar envíos en lotes cuando sea posible
    
- Usar conexiones persistentes (keep-alive)
    
- Implementar timeout apropiados (30-60 segundos)
    
- Monitorear tiempos de respuesta
    

### 9\. Manejo de Notas de Crédito y Débito

**Validaciones Especiales:**

- Verificar que el eCF original exista y esté aceptado
    
- Validar que el monto no exceda el original (NC)
    
- Mantener referencia clara al documento original
    
- Documentar claramente el motivo de la modificación
    
- Notificar al comprador sobre la modificación
    

**Proceso Recomendado:**

1. Consultar el eCF original en el sistema
    
2. Validar que no haya sido anulado previamente
    
3. Calcular el monto disponible para NC/ND
    
4. Generar la NC/ND con referencia correcta
    
5. Actualizar el saldo del documento original
    
6. Notificar a las partes involucradas
    

### 10\. Cumplimiento y Normativa

**Mantener Actualizado:**

- Revisar periódicamente actualizaciones de DGII
    
- Suscribirse a boletines informativos de DGII
    
- Participar en capacitaciones oficiales
    
- Mantener documentación actualizada
    
- Consultar con asesores fiscales ante dudas
    

**Recursos Oficiales:**

- Portal DGII: [https://dgii.gov.do](https://dgii.gov.do)
    
- Normas y Resoluciones: Revisar Norma 01-2019 y actualizaciones
    
- Soporte Técnico DGII: Contactar para casos específicos
    

---

## Respuesta del Sistema

### Estructura de Respuesta Exitosa

``` json
{
  "trackId": "4cdc9917-8008-4564-a85c-c3f13a4e0ec8",
  "codigo": "4",
  "estado": "Aceptado Condicional",
  "rnc": "101742186",
  "encf": "E320000000004",
  "url": "https://ecf.dgii.gov.do/testecf/consulta...",
  "codigoSeguridad": "EAkbyR",
  "fechaFirma": "15-11-2025 14:46:41",
  "secuenciaUtilizada": true,
  "fechaRecepcion": "2025-11-15T10:46:42",
  "mensajes": [
    {
      "valor": "El campo MontoGravadoI1 del área Totales...",
      "codigo": 1934
    }
  ]
}

 ```

### Campos de Respuesta

- **trackId**: Identificador único de la transacción para seguimiento
    
- **codigo**: Código de estado, como texto (ver [Códigos de Estado](#códigos-de-estado)). Para decidir, use `estado`
    
- **estado**: Descripción del estado
    
- **rnc**: RNC del emisor
    
- **encf**: Número del comprobante procesado
    
- **url**: URL para consulta pública del comprobante
    
- **codigoSeguridad**: Código de seguridad para validación
    
- **fechaFirma**: Fecha y hora de firma digital
    
- **secuenciaUtilizada**: Indica si la secuencia fue utilizada
    
- **fechaRecepcion**: Fecha y hora de recepción por DGII
    
- **mensajes**: Array de mensajes, advertencias o errores
    

---

## Casos de Uso Avanzados

### Caso 1: Factura con Múltiples Formas de Pago

``` json
{
  "encabezado": {
    "idDoc": {
      "tablaFormasPago": {
        "formaDePago": [
          {
            "formaPago": "1",
            "montoPago": "50000.00"
          },
          {
            "formaPago": "4",
            "montoPago": "68000.00"
          }
        ]
      }
    }
  }
}

 ```

### Caso 2: Factura con Descuentos y Recargos

``` json
{
  "encabezado": {
    "totales": {
      "MontoGravadoTotal": "100000.00",
      "TotalITBIS": "18000.00",
      "MontoDescuento": "5000.00",
      "MontoRecargo": "2000.00",
      "MontoTotal": "115000.00"
    }
  },
  "descuentosORecargos": {
    "descuentoORecargo": [
      {
        "tipo": 1,
        "descripcion": "Descuento por pronto pago",
        "monto": "5000.00",
        "indicadorNorma": 1
      },
      {
        "tipo": 2,
        "descripcion": "Recargo por entrega urgente",
        "monto": "2000.00",
        "indicadorNorma": 1
      }
    ]
  }
}

 ```

### Caso 3: Factura con Múltiples Tasas de ITBIS

``` json
{
  "encabezado": {
    "totales": {
      "MontoGravadoTotal": "150000.00",
      "MontoGravadoI1": "100000.00",
      "MontoGravadoI2": "50000.00",
      "MontoGravadoI3": "0.00",
      "ITBIS1": 18,
      "ITBIS2": 16,
      "TotalITBIS": "26000.00",
      "TotalITBIS1": "18000.00",
      "TotalITBIS2": "8000.00",
      "MontoTotal": "176000.00"
    }
  },
  "detallesItems": {
    "items": [
      {
        "numeroLinea": 1,
        "nombreItem": "Producto con ITBIS 18%",
        "cantidadItem": "1.00",
        "precioUnitarioItem": "100000.00",
        "montoItem": "100000.00"
      },
      {
        "numeroLinea": 2,
        "nombreItem": "Producto con ITBIS 16%",
        "cantidadItem": "1.00",
        "precioUnitarioItem": "50000.00",
        "montoItem": "50000.00"
      }
    ]
  }
}

 ```

---

## Preguntas Frecuentes (FAQ)

### ¿Puedo modificar un eCF después de enviado?

No. Una vez enviado y aceptado, un eCF no puede modificarse. Debe emitir una Nota de Crédito (E34) o Nota de Débito (E33) para hacer ajustes.

### ¿Qué hago si envío un eCF con error?

Si el eCF fue aceptado, debe emitir una Nota de Crédito (E34) para anularlo y luego emitir un nuevo eCF correcto.

### ¿Puedo reutilizar una secuencia si el envío falló?

Depende de cómo falló. Si no hubo respuesta, hubo un error de conexión o quedó pendiente, reenvíe con el **mismo** e-NCF: usar otro podría emitir dos facturas por la misma venta. Si la DGII lo rechazó, use uno **nuevo**, salvo que la respuesta traiga `secuenciaUtilizada: false`. Ver [qué hacer según la respuesta](#2-manejo-de-errores-y-reintentos).

### ¿Cuánto tiempo tengo para emitir una Nota de Crédito?

Según normativa DGII, las Notas de Crédito deben emitirse dentro del mismo período fiscal o según las políticas de la empresa, pero se recomienda hacerlo lo antes posible.

### ¿Qué tipo de eCF uso para un cliente extranjero?

- Si es exportación: E46
    
- Si es consumo local: E32 con identificadorextranjero
    

### ¿Cómo manejo productos exentos y gravados en la misma factura?

Incluya ambos tipos de ítems en el array de items, y asegúrese de que los totales reflejen correctamente MontoExento y MontoGravado.

### ¿Qué hago si recibo "Aceptado Condicional"?

El eCF es válido, pero revise los mensajes de advertencia. Corrija los problemas señalados para futuros envíos.

### ¿Puedo emitir eCF sin conexión a internet?

No directamente. Sin embargo, puede usar el indicadorEnvioDiferido para marcar eCF que se enviarán cuando haya conexión.

---

## Glosario de Términos

- **eCF**: Comprobante Fiscal Electrónico
    
- **DGII**: Dirección General de Impuestos Internos
    
- **RNC**: Registro Nacional de Contribuyentes
    
- **ITBIS**: Impuesto sobre la Transferencia de Bienes Industrializados y Servicios
    
- **NCF**: Número de Comprobante Fiscal (sistema anterior)
    
- **TrackId**: Identificador único de seguimiento asignado por DGII
    
- **Código de Seguridad**: Código alfanumérico para validación del eCF
    
- **Nota de Crédito**: Documento que reduce el valor de una factura
    
- **Nota de Débito**: Documento que aumenta el valor de una factura
    
- **Zona Franca**: Régimen especial de incentivos fiscales
    
- **Régimen Especial**: Tratamiento tributario diferenciado por sector o actividad
    

---

## Contacto y Soporte

Para soporte técnico o consultas sobre la implementación:

- **Portal DGII**: [https://dgii.gov.do](https://dgii.gov.do)
    
- **Soporte Técnico**: Contactar a través del portal oficial
    
- **Documentación Oficial**: Revisar Norma General 01-2019 y actualizaciones
    
- **Capacitaciones**: Consultar calendario de capacitaciones en portal DGII
    

---

**Última Actualización**: Mayo 2025  
**Versión del Documento**: 2.0  
**Versión de eCF Soportada**: 1.0
