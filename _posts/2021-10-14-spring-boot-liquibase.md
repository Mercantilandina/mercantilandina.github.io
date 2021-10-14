---
layout: post
title: Automatizar cambios en la base de datos con Liquibase
---

Publicar una nueva versión de una aplicación puede ser una tarea compleja si también requiere realizar cambios en la base de datos. Para simplificar y automatizar estos cambios podemos utilizar Liquibase.


## Configuración

Lo primero que necesitamos es agregar una dependencia en el archivo pom.xml


```xml
        <dependency>
            <groupId>org.liquibase</groupId>
            <artifactId>liquibase-core</artifactId>
        </dependency>
```


Con este cambio se ejecuta Liquibase de forma automática al iniciar la aplicación. 

Necesitamos hacer un ajuste más dado que por defecto el archivo principal de Liquibase lo busca en la siguiente ubicación: **db/changelog/db.changelog-master.yaml**. Podemos cambiar la ubicación en el archivo de configuración de Spring Boot application.yaml


```yaml
spring:

  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.xml
```


Utilizando Spring Boot, Liquibase se autoconfigura utilizando las mismas credenciales que la aplicación para acceder a la base de datos. Es necesario tener en cuenta que el usuario que utilicemos necesita tener permisos para modificar el esquema. 


## Changelog

El archivo db.changelog-master.xml indica a Liquibase donde se encuentran todos los script de cambios y en qué orden ejecutarlos. 


```xml
<?xml version="1.0" encoding="UTF-8"?>  
<databaseChangeLog  
  xmlns="http://www.liquibase.org/xml/ns/dbchangelog"  
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
  xsi:schemaLocation=
  "http://www.liquibase.org/xml/ns/dbchangelog
   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.8.xsd">  

  <include
   file="classpath:/db/changelog/changes/crear-tabla-usuario.xml" />
  <include
   file="classpath:/db/changelog/changes/columna-habilitado-usuario.xml" />
  <include
   file="classpath:/db/changelog/changes/datos-usuarios.sql" />
  <include
   file="classpath:/db/changelog/changes/crear-tabla-roles.yml" />
  <include
   file="classpath:/db/changelog/changes/crear-tabla-rolusuario.xml" />
  <include
   file="classpath:/db/changelog/changes/roles-usuarios.sql" />
</databaseChangeLog>
```


El orden de los archivos es importante: cada vez que necesitamos agregar un nuevo script, lo debemos incluir debajo de la lista. 


## Formato XML

Los cambios se pueden especificar en diferentes formatos. El más usado es el formato XML.

Siempre partimos de un elemento _databaseChangeLog_. Dentro incluimos los cambios con un elemento _changeSet_. Pueden incluirse varios, pero lo aconsejable es especificar un solo cambio por archivo. El elemento changeSet requiere de forma obligatoria indicar un id, que puede ser cualquier cadena de texto y un autor. 

En un nivel más abajo se indica el tipo de cambio, en el ejemplo mostramos un elemento _createTable_ para indicarle a Liquibase que debe crear una tabla. 


```xml
<?xml version="1.0" encoding="UTF-8"?>  
<databaseChangeLog  
  xmlns="http://www.liquibase.org/xml/ns/dbchangelog"  
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
  xsi:schemaLocation=
   "http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.8.xsd">  
 
  <changeSet id="crear tabla rolusuario" author="autor">
      <createTable tableName="rolusuario">
          <column name="usuario_id" type="char(36)">
              <constraints nullable="false"
                           foreignKeyName="fk_rolusuario_usuario_id"
                           referencedColumnNames="id"
                           referencedTableName="usuario"/>
          </column>
          <column name="rol_nombre" type="char(10)">
              <constraints nullable="false"
                           foreignKeyName="fk_rolusuario_rol_nombre"
                           referencedColumnNames="nombre"
                           referencedTableName="rol"/>
          </column>
      </createTable>
  </changeSet>                    
</databaseChangeLog>
```


En el siguiente ejemplo mostramos cómo especificar un cambio para agregar una columna


```xml
<?xml version="1.0" encoding="UTF-8"?>  
<databaseChangeLog  
  xmlns="http://www.liquibase.org/xml/ns/dbchangelog"  
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
  xsi:schemaLocation=
   "http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.8.xsd">  
 
  <changeSet id="agregar columna habilitado" author="yo">
      <addColumn tableName="usuario">
          <column name="habilitado"
                  type="boolean"
                  defaultValue="true" />
      </addColumn>
  </changeSet>                    
</databaseChangeLog>
```



## Formato SQL

Es posible utilizar SQL básico para especificar cambios en la base de datos. En el siguiente ejemplo mostramos que es posible insertar datos. 


```sql
--liquibase formatted sql

--changeset autor:datos-roles

insert into rol(nombre, descripcion)
value ('admin', 'Administrador del grupo');

insert into rol(nombre, descripcion)
value ('external', 'Usuario externo');

insert into rol(nombre, descripcion)
value ('invitado', 'Usuario invitado');

insert into rol(nombre, descripcion)
value ('owner', 'Usuario dueño del grupo');
```


Un registro de cambios con SQL debe comenzar con el siguiente comentario


```sql
--liquibase formatted sql
```


Luego debemos indicar el id del _changeset_ y el autor, con el siguiente formato: 


```sql
--changeset autor:datos-roles
```


Posteriormente incluimos las sentencias SQL terminadas por punto y coma.


## Formato YAML

Otro formato soportado es con una estructura de documento YAML. Los elementos son los mismos para el formato XML


```yaml
databaseChangeLog:
  - changeSet:
      id: 'crear tabla de roles'
      author: autor
      changes:
        - createTable:
            tableName: rol
            columns:
              - column:
                  name: nombre
                  type: char(50)
                  constraints:
                    primaryKey: true
                    nullable: false  
              - column:
                  name: descripcion
                  type: varchar(200)
                  defaultValue: ''
```

## Registro de cambios

Liquibase utiliza una tabla para registrar los cambios. La tabla tiene el nombre  DATABASECHANGELOG y se crea automáticamente si no existe. Esta tabla contiene la información de todos los cambios aplicados: el id del changeset, el autor, la fecha y hora de ejecución, entre otros datos.

También es importante tener en cuenta que esta tabla cuenta con un valor Checksum para validar que ningún changeset sea alterado una vez que se ejecutó. Por lo tanto, si hay que realizar un cambio, es necesario incluir un archivo nuevo. No se recomienda modificar los changeset aplicados porque da error y se interrumpe el proceso. 

## Ejemplo

Dejamos una aplicación de ejemplo en [Github](https://github.com/Mercantilandina/spring-boot-liquibase).


## Referencias
* [Documentación de Liquibase](https://docs.liquibase.com/concepts/home.html)
* [Liquibase con Spring Boot](https://docs.liquibase.com/tools-integrations/springboot/springboot.html)