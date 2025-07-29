# Guía de XSender

XSender es una librería de Java que permite enviar documentos electrónicos UBL (Universal Business Language) a la SUNAT (Superintendencia Nacional de Aduanas y de Administración Tributaria) de Perú. Esta guía proporciona una descripción detallada de todas las funcionalidades de XSender.

## 1. Visión General

XSender se encarga de la comunicación con los servicios web de la SUNAT para el envío de los siguientes documentos:

*   **Factura (Invoice)**
*   **Boleta de Venta**
*   **Nota de Crédito (CreditNote)**
*   **Nota de Débito (DebitNote)**
*   **Resumen Diario de Boletas**
*   **Comunicación de Baja**

Además del envío de documentos, XSender también permite:

*   **Consultar el estado de un ticket:** Para los envíos de resúmenes y comunicaciones de baja.
*   **Consultar la validez de un comprobante:** Para verificar si un comprobante es válido en los registros de la SUNAT.

La librería utiliza Apache Camel para gestionar las rutas y la comunicación con la SUNAT.

## 2. Envío de Documentos

Para enviar un documento UBL a la SUNAT, primero se debe crear un objeto `SunatRequest`. Este objeto contiene el archivo XML del documento y las credenciales de la empresa.

### 2.1. Ejemplo de Envío

El siguiente ejemplo muestra cómo enviar una factura a la SUNAT:

```java
import io.github.project.openubl.xsender.company.CompanyCredentials;
import io.github.project.openubl.xsender.company.CompanyURLs;
import io.github.project.openubl.xsender.sunat.SunatRequest;
import io.github.project.openubl.xsender.sunat.SunatResponse;
import io.github.project.openubl.xsender.sunat.BillServiceDestination;

import java.io.File;

public class Main {

    public static void main(String[] args) {
        // Credenciales de la empresa
        CompanyCredentials credentials = CompanyCredentials.builder()
                .ruc("12345678912")
                .username("MODDATOS")
                .password("MODDATOS")
                .build();

        // URLs de los servicios de la SUNAT
        CompanyURLs urls = CompanyURLs.builder()
                .invoice("https://e-beta.sunat.gob.pe/ol-ti-itcpfegem-beta/billService")
                .build();

        // Archivo XML de la factura
        File xmlFile = new File("factura.xml");

        // Crear la solicitud de envío
        SunatRequest request = SunatRequest.builder()
                .credentials(credentials)
                .xml(xmlFile)
                .destination(new BillServiceDestination(urls.getInvoice()))
                .build();

        // Enviar la solicitud y obtener la respuesta
        SunatResponse response = request.send();

        // Procesar la respuesta de la SUNAT
        if (response.getStatus() == Status.OK) {
            System.out.println("La factura ha sido aceptada.");
        } else {
            System.out.println("Error al enviar la factura: " + response.getError().getMessage());
        }
    }
}
```

## 3. Consulta de Tickets y Comprobantes

### 3.1. Consulta de Ticket

Para consultar el estado de un ticket, se debe utilizar el destino `BillConsultServiceDestination`:

```java
// ...
SunatRequest request = SunatRequest.builder()
        .credentials(credentials)
        .ticket("123456789") // Número de ticket
        .destination(new BillConsultServiceDestination(urls.getConsult()))
        .build();

SunatResponse response = request.send();
// ...
```

### 3.2. Consulta de Validez de Comprobante

Para consultar la validez de un comprobante, se debe utilizar el destino `BillValidServiceDestination`:

```java
// ...
SunatRequest request = SunatRequest.builder()
        .credentials(credentials)
        .document(SunatDocument.builder()
            .ruc("12345678912")
            .tipoComprobante("01")
            .serie("F001")
            .numero(1)
            .build()
        )
        .destination(new BillValidServiceDestination(urls.getValid()))
        .build();

SunatResponse response = request.send();
// ...
```

## 4. Configuración

### 4.1. Credenciales

Las credenciales de la empresa se configuran en el objeto `CompanyCredentials`. Se debe proporcionar el RUC, el usuario y la contraseña de la empresa.

### 4.2. URLs de la SUNAT

Las URLs de los servicios de la SUNAT se configuran en el objeto `CompanyURLs`. XSender proporciona URLs de prueba y producción, pero también se pueden especificar URLs personalizadas.

## 5. Respuestas de la SUNAT

XSender maneja las respuestas de la SUNAT y las encapsula en un objeto `SunatResponse`. Este objeto contiene el estado del envío, el CDR (Constancia de Recepción) y cualquier error que haya ocurrido.

### 5.1. Estado del Envío

El estado del envío se puede obtener a través del método `getStatus()` del objeto `SunatResponse`. Los posibles estados son:

*   **`OK`:** El documento fue aceptado por la SUNAT.
*   **`RECHAZADO`:** El documento fue rechazado por la SUNAT.
*   **`ERROR`:** Ocurrió un error durante el envío del documento.

### 5.2. CDR (Constancia de Recepción)

El CDR es un archivo XML que contiene la constancia de recepción del documento por parte de la SUNAT. Se puede obtener a través del método `getCdr()` del objeto `SunatResponse`.

### 5.3. Manejo de Errores

En caso de error, el objeto `SunatResponse` contendrá un objeto `SunatError` con información sobre el error. Se puede acceder a este objeto a través del método `getError()`.

## 6. Extensiones

XSender proporciona extensiones para facilitar su integración con los siguientes frameworks:

### 6.1. Quarkus

Para utilizar XSender en un proyecto de Quarkus, se debe agregar la siguiente dependencia al archivo `pom.xml`:

```xml
<dependency>
    <groupId>io.github.project-openubl</groupId>
    <artifactId>xsender-quarkus</artifactId>
    <version>${xsender.version}</version>
</dependency>
```

### 6.2. Spring Boot

Para utilizar XSender en un proyecto de Spring Boot, se debe agregar la siguiente dependencia al archivo `pom.xml`:

```xml
<dependency>
    <groupId>io.github.project-openubl</groupId>
    <artifactId>xsender-spring-boot</artifactId>
    <version>${xsender.version}</version>
</dependency>
```
