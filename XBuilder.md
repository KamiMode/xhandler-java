# Guía de XBuilder

XBuilder es una librería de Java que facilita la creación de documentos electrónicos UBL (Universal Business Language) para Perú. Esta guía proporciona una descripción detallada de todas las funcionalidades de XBuilder.

## 1. Visión General

XBuilder simplifica la generación de XML para los siguientes documentos UBL:

*   **Factura (Invoice):** Comprobante de venta que detalla una transacción comercial.
*   **Nota de Crédito (CreditNote):** Documento que anula o modifica una factura emitida anteriormente.
*   **Nota de Débito (DebitNote):** Documento que incrementa el importe de una factura emitida anteriormente.
*   **Resumen de Anulaciones (VoidedDocument):** Documento que se utiliza para anular boletas de venta o facturas.
*   **Resumen Diario (SummaryDocuments):** Documento que se utiliza para enviar un resumen de las boletas de venta emitidas en un día.
*   **Comprobante de Percepción (Perception):** Comprobante que emiten los agentes de percepción del IGV.
*   **Comprobante de Retención (Retention):** Comprobante que emiten los agentes de retención del IGV.
*   **Guía de Remisión (DespatchAdvice):** Documento que sustenta el traslado de bienes.

La librería está diseñada para ser fácil de usar y se integra con proyectos de Quarkus.

## 2. Creación de Documentos

Para crear un documento UBL, primero se debe crear un objeto del tipo de documento que se desea generar (por ejemplo, `Invoice`, `CreditNote`, o `DebitNote`). Estos objetos contienen toda la información necesaria para generar el XML.

### 2.1. Factura (Invoice)

El siguiente ejemplo muestra cómo crear una factura simple:

```java
import io.github.project.openubl.xbuilder.content.models.standard.general.Invoice;
import io.github.project.openubl.xbuilder.content.models.common.Cliente;
import io.github.project.openubl.xbuilder.content.models.common.Proveedor;
import io.github.project.openubl.xbuilder.content.models.standard.general.DocumentoVentaDetalle;

import java.math.BigDecimal;
import java.time.LocalDate;

public class Main {

    public static void main(String[] args) {
        Invoice invoice = Invoice.builder()
                .serie("F001")
                .numero(1)
                .fechaEmision(LocalDate.now())
                .proveedor(Proveedor.builder()
                        .ruc("12345678912")
                        .razonSocial("Mi Empresa S.A.C.")
                        .build()
                )
                .cliente(Cliente.builder()
                        .ruc("98765432198")
                        .razonSocial("Cliente S.A.")
                        .build()
                )
                .detalle(DocumentoVentaDetalle.builder()
                        .descripcion("Producto 1")
                        .cantidad(new BigDecimal("10"))
                        .precio(new BigDecimal("100"))
                        .build()
                )
                .build();

        // Generar el XML a partir del objeto Invoice
        // ...
    }
}
```

### 2.2. Otros Documentos

La creación de otros tipos de documentos sigue un patrón similar. Se debe crear un objeto del modelo correspondiente y llenar sus atributos. Por ejemplo, para crear una guía de remisión, se utilizaría el objeto `DespatchAdvice`.

## 3. Modelos de Datos

Los modelos de datos en XBuilder representan la estructura de los documentos UBL. Los modelos principales se encuentran en el paquete `io.github.project.openubl.xbuilder.content.models`.

### 3.1. Atributos Comunes

Algunos de los atributos más comunes en los modelos de datos son:

*   **`serie`:** La serie del comprobante (ej. "F001").
*   **`numero`:** El número del comprobante (ej. 1).
*   **`fechaEmision`:** La fecha de emisión del comprobante.
*   **`proveedor`:** Los datos del emisor del comprobante.
*   **`cliente`:** Los datos del receptor del comprobante.
*   **`detalle`:** La lista de productos o servicios incluidos en el comprobante.

### 3.2. Catálogos

XBuilder utiliza catálogos definidos por SUNAT para llenar ciertos campos de los documentos UBL. Estos catálogos se encuentran en el paquete `io.github.project.openubl.xbuilder.content.catalogs` y se utilizan para validar y autocompletar información.

El uso de los catálogos es automático en la mayoría de los casos. XBuilder utiliza la información proporcionada en los modelos de datos para seleccionar el valor correcto del catálogo.

Algunos de los catálogos más importantes son:

*   **`Catalog1`:** Tipos de documento de identidad.
*   **`Catalog6`:** Tipos de documento de identidad.
*   **`Catalog7`:** Tipos de afectación al IGV.
*   **`Catalog8`:** Tipos de sistema de cálculo del ISC.

## 4. Renderización de XML

XBuilder utiliza el motor de plantillas Qute para generar el XML de los documentos UBL. Las plantillas se encuentran en el directorio `src/main/resources/Renderer` y tienen la extensión `.xml`.

El proceso de renderización es el siguiente:

1.  Se crea un objeto del modelo de datos (ej. `Invoice`).
2.  Se pasa el objeto al motor de plantillas Qute.
3.  Qute utiliza la plantilla correspondiente al tipo de documento para generar el XML.

Este enfoque permite modificar las plantillas XML si es necesario, sin tener que modificar el código Java.

## 5. Extensión de Quarkus

XBuilder proporciona una extensión de Quarkus que facilita su integración en aplicaciones Quarkus. Para utilizar la extensión, se debe agregar la siguiente dependencia al archivo `pom.xml`:

```xml
<dependency>
    <groupId>io.github.project-openubl</groupId>
    <artifactId>xbuilder-quarkus</artifactId>
    <version>${xbuilder.version}</version>
</dependency>
```

La extensión de Quarkus configura automáticamente los beans necesarios para utilizar XBuilder en la aplicación.
