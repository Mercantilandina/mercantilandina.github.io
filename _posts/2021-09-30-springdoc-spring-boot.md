---
layout: post
title: Generar documentación de las APIs con Springdoc
---

OpenAPI es un estándar de documentación de servicios Rest y está basado originalmente en la especificación de Swagger. Podemos generar documentos que sigan este estándar en nuestras aplicaciones de Spring Boot utilizando la librería Springdoc.


## Incluir SpringDoc en el proyecto

Incluimos la librería Springdoc en las dependencias del proyecto Maven, en el archivo _pom.xml_.


```xml
  <dependency>
      <groupId>org.springdoc</groupId>
      <artifactId>springdoc-openapi-ui</artifactId>
      <version>1.5.10</version>
   </dependency>
```


A partir de aquí ya se activa el generador de documentación con la configuración básica. Si ejecutamos la aplicación podemos acceder al documento en la siguiente dirección:

http://localhost:8080/swagger-ui.html

![Caputa de pantalla](https://raw.githubusercontent.com/Mercantilandina/springdoc-example/master/docs/2021-09-29_223550.png)

## Encabezado del documento

OpenApi posee una serie de atributos que se exponen al inicio del documento y representan la información general de la API. El encabezado se compone de un objeto _info_ con la descripción y _servers_ es la enumeración de las direcciones web de los servicios. 

Para definir estos valores podemos generar un bean de configuración tal como se muestra en este ejemplo:


```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.servers.Server;

@Configuration
public class OpenAPIConfig {

    @Bean
    public OpenAPI config() {
        return new OpenAPI()
            .info(new Info()
                    .title("API de usuarios")
                    .description("Servicios para gestionar usuarios")
                    .version("1.0.5"))
            .addServersItem(new Server()
                                .description("Producción")
                                .url("https://users.acme.com"))
            .addServersItem(new Server()
                                .description("Servicio de QA")
                                .url("https://users.qa.acme.com"));
    }

}
```

## Rutas

Por defecto esta librería escanea todos los controladores para catalogar los servicios y sus rutas. La información que se obtiene se puede enriquecer mediante anotaciones. 

La anotación más importante es _@Operation_ que permite describir el servicio. El atributo _summary _contiene un resumen de la funcionalidad. Con _description_ podemos ampliar la definición del servicio.

Además un servicio puede tener una o más tipo de respuestas; estas se especifican con la anotación _@ApiResponses_. Indicamos una descripción, el código de estado y contenido. En la descripción del contenido se puede incluir varios ejemplos que pueden ilustrar los distintos tipos de respuesta.

```java
public interface ConsultarUsuario {

    @Operation(
        summary = "Consulta un usuario",
        description = "Consultar los datos de un usuario " +
                      " ingresando como parámetro el identificador"
    )
    @ApiResponses(
        value = {
            @ApiResponse(
                description = "Datos del usuario",
                responseCode = "200",
                content = {
                    @Content(
                        mediaType = "application/json",
                        schema = @Schema(
                            implementation = UsuarioResponse.class
                        ),
                        examples = {
                            @ExampleObject(
                                name = "Usuario activo",
                                value = "{\"id\": \"23\", " + 
                                        "\"nombre\": \"user\", " + 
                                        "\"email\": " + 
                                        " \"user@mail.com\", " + 
                                        " \"activo\": true}"
                            )
                        }    
                    )
                }
            ),
            @ApiResponse(
                description = "Usuario no existe",
                responseCode = "404",
                content = @Content
            )
        }
    )
    @GetMapping("/usuarios/{id}")
    UsuarioResponse consultar(
        @Parameter(
            description = "Identificador de usuario",
            example = "123",
            required = true
        )
        @PathVariable String id
    );

}
```


Para organizar mejor nuestro código, y que quede más legible, es posible documentar una interfaz y luego implementarla en un controlador.


## Esquemas

Cada estructura de datos que componen los servicios se pueden anotar para extender la documentación con una descripción de cada uno de los atributos. La anotación que se utiliza es @Schema y se agrega en cada atributo de la clase. 


```java
import io.swagger.v3.oas.annotations.media.Schema;

public class UsuarioResponse {
    @Schema(description = "Identificador de usuario")
    private String id;

    @Schema(description = "Nombre de usuario")
    private String nombre;

    @Schema(description = "Correo electrónico")
    private String email;

    @Schema(description = "Indica si el usuario está activo")
    private boolean activo;
}
```


Para generar la documentación de los esquemas también se toman las anotaciones de la especificación de Bean Validation.


```java
public class UsuarioRequest {

    @Schema(description = "Identificador de usuario", example = "123")
    @NotEmpty(message = "Ingresar el id de usuario")
    @Size(min = 1, max = 16)
    private String id;

    @Schema(description = "Nombre de usuario", example = "user")
    @NotEmpty(message = "Ingresar el nombre de usuario")
    @Size(min = 1, max = 100)
    private String nombre;

    @Schema(description = "Correo electrónico", 
            example = "user@mail.com", format = "email")
    @Email
    @NotEmpty(message = "Ingresar el email de usuario")
    @Size(min = 1, max = 120)
    private String email;

    @Schema(description = "Indica si el usuario está activo", 
            example = "true")
    @NotNull
    private boolean activo;
}
```



## Etiquetas

Cada servicio puede pertenecer a una categoría. Para crear estas categorías o etiquetas podemos utilizar la configuración del documento con el método _addTagsItem_ para incluir un objeto de la clase _Tag_. Se pueden crear tantas etiquetas como sea necesario. Los atributos básicos de una etiqueta son el nombre y una descripción.


```java
@Configuration
public class OpenAPIConfig {
    @Bean
    public OpenAPI config() {
        return new OpenAPI()
            .info(new Info()
                    .title("API de usuarios")
                    .description("Servicios de usuarios del sistema")
                    .version("1.0.5"))
            .addTagsItem(new Tag()
                    .name("usuarios")
                    .description("Servicios de usuario"));

    }
    
}
```


Una vez creada la etiqueta es posible asignarla a los servicios. Para asociar un servicio con una o más etiquetas se indican en el atributo _tags_ de la anotación _@Operation_. 


```java
public interface CrearUsuario {
 
    @Operation(
        summary = "Crear usuario",
        description = "Se crea un nuevo usuario",
        tags = {"usuarios"}
    )
    void crear(@RequestBody @Valid UsuarioRequest request);
```



## Plugin para generar el documento

Normalmente estos documentos se generan cuando la aplicación está ejecutándose. Sin embargo es posible generarla cuando se compila, utilizando un plugin de Maven.


```xml
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <jvmArguments>
                    -Dspring.application.admin.enabled=true
                </jvmArguments>
            </configuration>
            <executions>
                <execution>
                    <id>pre-integration-test</id>
                    <goals>
                        <goal>start</goal>
                    </goals>
                </execution>
                <execution>
                    <id>post-integration-test</id>
                    <goals>
                        <goal>stop</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        <plugin>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-maven-plugin</artifactId>
            <version>1.3</version>
            <executions>
                <execution>
                    <id>integration-test</id>
                    <goals>
                        <goal>generate</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
```


Es necesario modificar la configuración de Spring para que la aplicación se ejecute después la compilación y así de esta manera obtener la documentación generada. 

Para generar el archivo solo es necesario ejecutar este comando:


```bash
mvn verify
```



## Ejemplo

Dejamos una aplicación de ejemplo en [Github](https://github.com/Mercantilandina/springdoc-example).

## Referencias

* [Plugin de Maven](https://github.com/springdoc/springdoc-openapi-maven-plugin)
* [Documentación de Springdoc](https://springdoc.org/)
* [Especificación de OpenAPI](https://spec.openapis.org/oas/latest.html)