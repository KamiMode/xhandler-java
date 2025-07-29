# Guía Detallada para la Creación y Envío de Documentos Electrónicos a la SUNAT con XBuilder y XSender

Esta guía proporciona una explicación completa y detallada sobre cómo utilizar las librerías **XBuilder** y **XSender** para la generación, firma y envío de Comprobantes de Pago Electrónicos (CPE) a la Superintendencia Nacional de Aduanas y de Administración Tributaria (SUNAT) en Perú.

El objetivo es que los desarrolladores puedan integrar estas herramientas en sus sistemas de facturación de manera eficiente, comprendiendo a fondo cada paso del proceso y aprovechando al máximo las funcionalidades que ofrecen.

## ¿Por qué usar XBuilder y XSender?

- **XBuilder**: Simplifica la creación de los XML para los CPE. Se encarga de la estructura del documento, el cumplimiento de los catálogos de la SUNAT y los cálculos automáticos de impuestos y totales, permitiéndote enfocarte en los datos de negocio.
- **XSender**: Gestiona la comunicación con los web services de la SUNAT. Maneja el envío de los documentos, la consulta de tickets y la interpretación de las respuestas, abstrayendo la complejidad de la comunicación directa con la SUNAT.

---

## Tabla de Contenidos

1.  [Configuración del Proyecto](#1-configuración-del-proyecto)
2.  [Documentos Soportados](#2-documentos-soportados)
3.  [Creación de Documentos con XBuilder](#3-creación-de-documentos-con-xbuilder)
    -   [Factura y Boleta de Venta](#31-factura-y-boleta-de-venta)
    -   [Nota de Crédito](#32-nota-de-crédito)
    -   [Nota de Débito](#33-nota-de-débito)
    -   [Baja de Documentos (VoidedDocuments)](#34-baja-de-documentos-voideddocuments)
    -   [Resumen Diario (SummaryDocuments)](#35-resumen-diario-summarydocuments)
    -   [Percepción](#36-percepción)
    -   [Retención](#37-retención)
    -   [Guía de Remisión](#38-guía-de-remisión)
4.  [Firma Digital del XML](#4-firma-digital-del-xml)
5.  [Envío de Documentos con XSender](#5-envío-de-documentos-con-xsender)
    -   [Envío de Facturas, Boletas, Notas de Crédito y Débito](#51-envío-de-facturas-boletas-notas-de-crédito-y-débito)
    -   [Envío de Bajas y Resúmenes (Proceso Asíncrono)](#52-envío-de-bajas-y-resúmenes-proceso-asíncrono)
    -   [Envío de Percepciones y Retenciones](#53-envío-de-percepciones-y-retenciones)
    -   [Envío de Guías de Remisión](#54-envío-de-guías-de-remisión)
6.  [Manejo de Errores Comunes](#6-manejo-de-errores-comunes)

---

## 1. Configuración del Proyecto

Para empezar, necesitas un proyecto Java (se recomienda Maven o Gradle) y agregar las dependencias de XBuilder y XSender.

### Dependencias (Maven)

```xml
<dependencies>
    <!-- XBuilder para crear los XML (incluye firma) -->
    <dependency>
        <groupId>io.github.project-openubl</groupId>
        <artifactId>xbuilder</artifactId>
        <version>5.0.3-SNAPSHOT</version>
    </dependency>

    <!-- XSender para enviar los documentos a la SUNAT -->
    <dependency>
        <groupId>io.github.project-openubl</groupId>
        <artifactId>xsender</artifactId>
        <version>5.0.3-SNAPSHOT</version>
    </dependency>
</dependencies>
```

---

## 2. Documentos Soportados

Esta guía cubre la creación y envío de los siguientes documentos:

- **Factura** (`Invoice`): Para transacciones comerciales con clientes que tienen RUC.
- **Boleta de Venta** (`Invoice`): Para consumidores finales (DNI u otros).
- **Nota de Crédito** (`CreditNote`): Para anular o corregir una Factura o Boleta.
- **Nota de Débito** (`DebitNote`): Para incrementar el valor de una Factura o Boleta.
- **Baja de Documentos** (`VoidedDocuments`): Para anular Facturas o Boletas ya aceptadas por SUNAT.
- **Resumen Diario** (`SummaryDocuments`): Para informar sobre Boletas y sus notas asociadas.
- **Percepción** (`Perception`): Comprobante de percepción del IGV.
- **Retención** (`Retention`): Comprobante de retención del IGV.
- **Guía de Remisión** (`Despatch`): Para sustentar el traslado de mercancías.

---

## 3. Creación de Documentos con XBuilder

XBuilder utiliza un patrón de diseño "builder" para facilitar la creación de los objetos que representan cada documento.

### Configuración General de XBuilder

Antes de crear los documentos, es útil definir una configuración por defecto que se aplicará a todos ellos.

```java
import io.github.project.openubl.xbuilder.enricher.config.Defaults;
import io.github.project.openubl.xbuilder.enricher.config.DateProvider;
import java.math.BigDecimal;
import java.time.LocalDate;

// Tasas de impuestos y configuración por defecto
Defaults defaults = Defaults.builder()
        .icbTasa(new BigDecimal("0.50")) // Tasa del Impuesto al Consumo de Bolsas Plásticas (ej. 2023)
        .igvTasa(new BigDecimal("0.18")) // Tasa del IGV
        .build();

// Proveedor de fechas (puedes usar la fecha actual o una fecha fija para pruebas)
DateProvider dateProvider = () -> LocalDate.now();

// El ContentEnricher aplicará la configuración y realizará cálculos automáticos
ContentEnricher enricher = new ContentEnricher(defaults, dateProvider);
```

### 3.1. Factura y Boleta de Venta (`Invoice`)

Ambos documentos se crean con el objeto `Invoice`. La principal diferencia es la información del cliente y la serie del comprobante.

**Ejemplo de Factura:**

```java
import io.github.project.openubl.xbuilder.content.catalogs.Catalog6;
import io.github.project.openubl.xbuilder.content.models.common.Cliente;
import io.github.project.openubl.xbuilder.content.models.common.Proveedor;
import io.github.project.openubl.xbuilder.content.models.standard.general.DocumentoVentaDetalle;
import io.github.project.openubl.xbuilder.content.models.standard.general.Invoice;
import io.github.project.openubl.xbuilder.renderer.TemplateProducer;

// ... (usar la configuración de 'defaults' y 'enricher' definida antes)

Invoice invoice = Invoice.builder()
    .serie("F001")
    .numero(1)
    .proveedor(Proveedor.builder()
        .ruc("12345678912")
        .razonSocial("Mi Empresa S.A.C.")
        .build())
    .cliente(Cliente.builder()
        .nombre("Cliente de Ejemplo S.A.")
        .numeroDocumentoIdentidad("20123456789")
        .tipoDocumentoIdentidad(Catalog6.RUC.toString()) // RUC para Factura
        .build())
    .detalle(DocumentoVentaDetalle.builder()
        .descripcion("Laptop Gamer XYZ")
        .cantidad(new BigDecimal("2"))
        .precio(new BigDecimal("5000")) // Precio sin IGV
        .build())
    .detalle(DocumentoVentaDetalle.builder()
        .descripcion("Monitor Curvo 4K")
        .cantidad(new BigDecimal("3"))
        .precio(new BigDecimal("1500")) // Precio sin IGV
        .build())
    .build();

// 1. Enriquecer el documento (cálculos de IGV, totales, etc.)
enricher.enrich(invoice);

// 2. Generar el XML a partir del objeto
TemplateProducer template = TemplateProducer.getInstance();
String xmlSinFirmar = template.getInvoice().data(invoice).render();

System.out.println("--- FACTURA XML (SIN FIRMAR) ---");
System.out.println(xmlSinFirmar);
```

**Para crear una Boleta de Venta:**

- Cambia la `serie` (ej. "B001").
- En `cliente`, usa `tipoDocumentoIdentidad` como `Catalog6.DNI` u otro tipo para consumidor final.
- El `numeroDocumentoIdentidad` debe corresponder al tipo. Si el monto es menor a S/ 700 y el cliente no se identifica, se puede usar un DNI genérico "00000000".

---

### 3.2. Nota de Crédito (`CreditNote`)

Se usa para anular o descontar valor a una factura o boleta ya emitida.

```java
import io.github.project.openubl.xbuilder.content.catalogs.Catalog9;
// ... otros imports

CreditNote creditNote = CreditNote.builder()
    .serie("FC01") // Serie que identifica a las notas de crédito de facturas
    .numero(1)
    .comprobanteAfectadoSerieNumero("F001-1") // Factura que se está corrigiendo
    .sustento("Devolución del producto por falla de fábrica")
    .tipoNotaCredito(Catalog9.ANULACION_DE_LA_OPERACION.getCode())
    .proveedor(/* ... igual que la factura ... */)
    .cliente(/* ... igual que la factura ... */)
    .detalle(DocumentoVentaDetalle.builder()
        .descripcion("Laptop Gamer XYZ")
        .cantidad(new BigDecimal("1")) // Se devuelve 1 de las 2 compradas
        .precio(new BigDecimal("5000"))
        .build())
    .build();

enricher.enrich(creditNote);
String xmlNotaCredito = TemplateProducer.getInstance().getCreditNote().data(creditNote).render();
```

---

### 3.3. Nota de Débito (`DebitNote`)

Se usa para aumentar el valor de una factura o boleta (ej. por intereses o gastos de envío no facturados).

```java
import io.github.project.openubl.xbuilder.content.catalogs.Catalog10;
// ... otros imports

DebitNote debitNote = DebitNote.builder()
    .serie("FD01")
    .numero(1)
    .comprobanteAfectadoSerieNumero("F001-1")
    .sustento("Intereses por pago tardío")
    .tipoNotaDebito(Catalog10.INTERESES_POR_MORA.getCode())
    .proveedor(/* ... */)
    .cliente(/* ... */)
    .detalle(DocumentoVentaDetalle.builder()
        .descripcion("Intereses por mora")
        .cantidad(new BigDecimal("1"))
        .precio(new BigDecimal("100")) // Monto del interés
        .build())
    .build();

enricher.enrich(debitNote);
String xmlNotaDebito = TemplateProducer.getInstance().getDebitNote().data(debitNote).render();
```

---

### 3.4. Baja de Documentos (`VoidedDocuments`)

Para anular **Facturas** o **Boletas** que fueron previamente aceptadas por SUNAT. Este es un proceso asíncrono.

```java
import io.github.project.openubl.xbuilder.content.models.sunat.baja.VoidedDocuments;
import io.github.project.openubl.xbuilder.content.models.sunat.baja.VoidedDocumentsItem;

VoidedDocuments voidedDocuments = VoidedDocuments.builder()
    .numero(1) // Correlativo de bajas (empieza en 1 cada día)
    .proveedor(/* ... */)
    .baja(VoidedDocumentsItem.builder()
        .tipoComprobante("01") // 01: Factura
        .serie("F001")
        .numero(123)
        .descripcion("Se canceló la venta")
        .build())
    .baja(VoidedDocumentsItem.builder()
        .tipoComprobante("01")
        .serie("F001")
        .numero(124)
        .descripcion("Error en los datos del cliente")
        .build())
    .build();

enricher.enrich(voidedDocuments);
String xmlBaja = TemplateProducer.getInstance().getVoidedDocuments().data(voidedDocuments).render();
```

---

### 3.5. Resumen Diario (`SummaryDocuments`)

Para informar a SUNAT sobre las **Boletas de Venta** y sus notas asociadas emitidas durante un día. También es un proceso asíncrono.

```java
import io.github.project.openubl.xbuilder.content.models.sunat.resumen.SummaryDocuments;
import io.github.project.openubl.xbuilder.content.models.sunat.resumen.SummaryDocumentsItem;
import io.github.project.openubl.xbuilder.content.catalogs.Catalog1;

SummaryDocuments summary = SummaryDocuments.builder()
    .numero(1) // Correlativo de resúmenes (empieza en 1 cada día)
    .proveedor(/* ... */)
    .resumen(SummaryDocumentsItem.builder()
        .tipoComprobante("03") // Boleta
        .serie("B001")
        .numero(1)
        .estado(SummaryDocumentsItem.Estado.ADICIONAR) // Se está agregando esta boleta
        .clienteTipoDocumento(Catalog6.DNI.getCode())
        .clienteNumeroDocumento("12345678")
        .total(new BigDecimal("118.00")) // Monto total de la boleta
        .igv(new BigDecimal("18.00"))
        .build())
    .resumen(SummaryDocumentsItem.builder()
        .tipoComprobante("07") // Nota de Crédito de Boleta
        .serie("BC01")
        .numero(1)
        .estado(SummaryDocumentsItem.Estado.ADICIONAR)
        .comprobanteAfectadoTipo("03") // Afecta a una Boleta
        .comprobanteAfectadoSerieNumero("B001-5")
        .clienteTipoDocumento(Catalog6.DNI.getCode())
        .clienteNumeroDocumento("87654321")
        .total(new BigDecimal("59.00"))
        .igv(new BigDecimal("9.00"))
        .build())
    .build();

enricher.enrich(summary);
String xmlResumen = TemplateProducer.getInstance().getSummaryDocuments().data(summary).render();
```

---

### 3.6. Percepción (`Perception`)

Comprobante de percepción del IGV, emitido por un Agente de Percepción.

```java
import io.github.project.openubl.xbuilder.content.models.sunat.percepcion.Perception;
// ... otros imports

Perception perception = Perception.builder()
    .serie("P001")
    .numero(1)
    .proveedor(/* ... Agente de percepción ... */)
    .cliente(/* ... Cliente al que se le percibe ... */)
    .tasa(new BigDecimal("0.02")) // Tasa de percepción (ej. 2%)
    .importeTotalPercibido(new BigDecimal("20.00"))
    .importeTotalCobrado(new BigDecimal("1020.00"))
    .comprobante(PerceptionComprobante.builder()
        .tipoComprobante("01") // Factura
        .serie("F001")
        .numero("456")
        .fechaEmision(LocalDate.of(2023, 10, 26))
        .importeTotal(new BigDecimal("1000"))
        .moneda("PEN")
        .importeCobrado(new BigDecimal("1020"))
        .fechaCobro(LocalDate.now())
        .build())
    .build();

enricher.enrich(perception);
String xmlPercepcion = TemplateProducer.getInstance().getPerception().data(perception).render();
```

---

### 3.7. Retención (`Retention`)

Comprobante de retención del IGV, emitido por un Agente de Retención.

```java
import io.github.project.openubl.xbuilder.content.models.sunat.retencion.Retention;
// ... otros imports

Retention retention = Retention.builder()
    .serie("R001")
    .numero(1)
    .proveedor(/* ... Agente de retención ... */)
    .cliente(/* ... Proveedor al que se le retiene ... */)
    .tasa(new BigDecimal("0.03")) // Tasa de retención (ej. 3%)
    .importeTotalRetenido(new BigDecimal("30.00"))
    .importeTotalPagado(new BigDecimal("970.00"))
    .comprobante(RetentionComprobante.builder()
        .tipoComprobante("01") // Factura
        .serie("F002")
        .numero("789")
        .fechaEmision(LocalDate.of(2023, 10, 25))
        .importeTotal(new BigDecimal("1000"))
        .moneda("PEN")
        .importePagado(new BigDecimal("970"))
        .fechaPago(LocalDate.now())
        .build())
    .build();

enricher.enrich(retention);
String xmlRetencion = TemplateProducer.getInstance().getRetention().data(retention).render();
```

---

### 3.8. Guía de Remisión (`Despatch`)

Documento que sustenta el traslado de bienes.

```java
import io.github.project.openubl.xbuilder.content.models.sunat.guia.Despatch;
// ... otros imports

Despatch despatch = Despatch.builder()
    .serie("T001")
    .numero(1)
    .proveedor(/* ... Remitente ... */)
    .destinatario(/* ... Destinatario de los bienes ... */)
    .envio(Envio.builder()
        .motivoTraslado(Catalog20.VENTA.getCode())
        .descripcionMotivoTraslado("Venta de mercadería")
        .pesoBrutoTotal(new BigDecimal("150.5"))
        .unidadMedida("KGM")
        .partida(Partida.builder()
            .ubigeo("150101")
            .direccion("Av. Principal 123, Lima")
            .build())
        .llegada(Partida.builder()
            .ubigeo("040101")
            .direccion("Av. Sol 456, Arequipa")
            .build())
        .transportista(/* ... Datos del transportista ... */)
        .build())
    .bien(DespatchItem.builder()
        .cantidad(new BigDecimal("100"))
        .descripcion("Cajas de Zapatos Modelo X")
        .unidadMedida("NIU")
        .build())
    .build();

enricher.enrich(despatch);
String xmlGuia = TemplateProducer.getInstance().getDespatch().data(despatch).render();
```

---

## 4. Firma Digital del XML

Antes de ser enviado a la SUNAT, el XML debe ser firmado digitalmente con un Certificado Digital válido.

```java
import io.github.project.openubl.xbuilder.signature.XMLSigner;
import io.github.project.openubl.xbuilder.signature.CertificateDetails;
import io.github.project.openubl.xbuilder.signature.CertificateDetailsFactory;
import org.w3c.dom.Document;
import java.io.InputStream;
import java.security.PrivateKey;
import java.security.cert.X509Certificate;

public Document firmarXml(String xmlSinFirmar, String keystorePath, String keystorePassword, String keystoreAlias) throws Exception {
    // Cargar el certificado desde un archivo .pfx o .jks
    InputStream ksInputStream = getClass().getClassLoader().getResourceAsStream(keystorePath);

    // Obtener los detalles del certificado
    CertificateDetails certificateDetails = CertificateDetailsFactory.create(ksInputStream, keystorePassword);

    X509Certificate certificate = certificateDetails.getX509Certificate();
    PrivateKey privateKey = certificateDetails.getPrivateKey();

    // Firmar el documento XML
    return XMLSigner.signXML(xmlSinFirmar, "openubl", certificate, privateKey);
}
```

**Nota:** El `Document` firmado debe ser convertido a un `byte[]` o `String` para guardarlo o enviarlo.

---

## 5. Envío de Documentos con XSender

Una vez que tienes el XML firmado, XSender se encarga de enviarlo al endpoint correcto de la SUNAT.

### Configuración de XSender

```java
import io.github.project.openubl.xsender.company.CompanyCredentials;
import io.github.project.openubl.xsender.company.CompanyURLs;
import io.github.project.openubl.xsender.XSender;

// URLs de los servicios de la SUNAT (beta o producción)
CompanyURLs companyURLs = CompanyURLs.builder()
    .invoice("https://e-beta.sunat.gob.pe/ol-ti-itcpfegem-beta/billService")
    .perceptionRetention("https://e-beta.sunat.gob.pe/ol-ti-itemision-otroscpe-gem-beta/billService")
    .despatch("https://api-cpe.sunat.gob.pe/v1/contribuyente/gem")
    .build();

// Credenciales del Usuario SOL (secundario)
CompanyCredentials credentials = CompanyCredentials.builder()
    .username("TU_USUARIO_SOL") // Generalmente RUC + Usuario
    .password("TU_CLAVE_SOL")
    .build();

// Crear una instancia de XSender
XSender sender = new XSender(credentials, companyURLs);
```

### 5.1. Envío de Facturas, Boletas, Notas de Crédito y Débito

Estos documentos se envían al `billService` y la respuesta es síncrona (aceptado o rechazado al instante).

```java
import io.github.project.openubl.xsender.models.SunatResponse;
import java.io.File; // o byte[]

public void enviarFactura(File xmlFirmado) {
    try {
        // XSender analiza el XML para determinar a qué endpoint enviarlo
        SunatResponse response = sender.send(xmlFirmado);

        if (response.getStatus() == SunatResponse.Status.ACEPTADO) {
            System.out.println("Documento ACEPTADO.");
            // Guardar el CDR (Constancia de Recepción)
            response.getCdr().ifPresent(cdr -> {
                // cdr es un byte[] que puedes guardar como .zip
            });
        } else if (response.getStatus() == SunatResponse.Status.RECHAZADO) {
            System.out.println("Documento RECHAZADO.");
            response.getError().ifPresent(error -> {
                System.out.println("Código de error: " + error.getCode());
                System.out.println("Descripción: " + error.getDescription());
            });
        } else {
            System.out.println("El estado es: " + response.getStatus());
        }

    } catch (Exception e) {
        // Manejar excepciones de conexión, etc.
        e.printStackTrace();
    }
}
```

### 5.2. Envío de Bajas y Resúmenes (Proceso Asíncrono)

El envío de `VoidedDocuments` y `SummaryDocuments` devuelve un **ticket**. Debes consultar este ticket para saber el resultado final.

```java
public void enviarBaja(File xmlBaja) {
    try {
        SunatResponse response = sender.send(xmlBaja);

        if (response.getTicket().isPresent()) {
            String ticket = response.getTicket().get();
            System.out.println("Baja enviada. Ticket: " + ticket);

            // Esperar unos segundos antes de consultar el ticket
            Thread.sleep(5000); // 5 segundos de espera (en producción puede ser más)

            // Consultar el ticket
            SunatResponse ticketResponse = sender.verify(xmlBaja, ticket);

            if (ticketResponse.getStatus() == SunatResponse.Status.ACEPTADO) {
                System.out.println("La baja fue procesada y ACEPTADA.");
                ticketResponse.getCdr().ifPresent(cdr -> { /* ... guardar CDR ... */ });
            } else {
                System.out.println("La baja fue procesada con el estado: " + ticketResponse.getStatus());
                ticketResponse.getError().ifPresent(error -> {
                    System.out.println("Error: " + error.getDescription());
                });
            }
        } else {
            // Error en el envío inicial
            System.out.println("No se pudo enviar la baja.");
            response.getError().ifPresent(error -> System.out.println("Error: " + error.getDescription()));
        }

    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

### 5.3. Envío de Percepciones y Retenciones

Estos documentos usan un endpoint diferente, pero XSender lo maneja automáticamente. El proceso es síncrono.

```java
// El código de envío es idéntico al de una factura
public void enviarPercepcion(File xmlPercepcion) {
    SunatResponse response = sender.send(xmlPercepcion);
    // Procesar la respuesta igual que en la sección 5.1
}
```

### 5.4. Envío de Guías de Remisión

Las guías de remisión usan una API REST más moderna de SUNAT. XSender también abstrae esta complejidad.

```java
// El código de envío es idéntico al de una factura
public void enviarGuia(File xmlGuia) {
    SunatResponse response = sender.send(xmlGuia);
    // Procesar la respuesta igual que en la sección 5.1
}
```

---

## 6. Manejo de Errores Comunes

Durante el envío a la SUNAT, puedes encontrar diversos errores. Aquí algunos de los más comunes y cómo interpretarlos:

- **Código `2109`**: "El comprobante fue registrado previamente con otro ticket".
  - **Causa**: Estás intentando enviar un documento (ej. resumen) que ya fue enviado y está siendo procesado.
  - **Solución**: No reintentar. Consulta el estado del envío original si es posible.

- **Código `2235`**: "El nombre o razón social del emisor no coincide con el registrado en el RUC".
  - **Causa**: La razón social en tu XML no es exactamente igual a la que SUNAT tiene registrada para tu RUC.
  - **Solución**: Verifica en tu ficha RUC cuál es tu razón social exacta y úsala en el campo `proveedor.razonSocial`.

- **Código `2333`**: "El comprobante de pago electrónico fue emitido en una fecha anterior a la fecha de baja".
  - **Causa**: Estás intentando dar de baja un comprobante con una fecha de emisión posterior a la fecha de la comunicación de baja.
  - **Solución**: La fecha de la baja debe ser igual or posterior a la fecha de emisión del documento a anular.

- **Errores de Conexión (`java.net.ConnectException`)**:
  - **Causa**: Problemas de red, firewall, o el servicio de la SUNAT no está disponible.
  - **Solución**: Revisa tu conexión a internet y la configuración de tu firewall. Si el problema persiste, es probable que sea una incidencia de la SUNAT.

- **Errores de Autenticación (`401 Unauthorized`)**:
  - **Causa**: Las credenciales SOL (usuario y contraseña) son incorrectas.
  - **Solución**: Verifica que estás usando un usuario secundario SOL con los permisos necesarios para el envío de CPE y que las credenciales son correctas.

---

## Conclusión

Esta guía ha cubierto en profundidad el proceso de creación, firma y envío de documentos electrónicos a la SUNAT utilizando XBuilder y XSender. Para casos más complejos, como el manejo de diferentes tipos de impuestos, descuentos globales, o campos opcionales, te recomendamos consultar la documentación oficial de OpenUBL y los catálogos y normativas vigentes de la SUNAT.
