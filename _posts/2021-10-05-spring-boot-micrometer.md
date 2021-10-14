---
layout: post
title: Crear métricas con Prometheus y Micrometer
---

Cuando desplegamos microservicios es muy importante obtener métricas que nos permitan en todo momento verificar que la aplicación tiene el comportamiento que esperamos. Para ello Micromenter nos ofrece por defecto una serie de métricas de base que nos permiten observar el estado de salud del servicio. Pero además podemos complementarlas con métricas personalizadas. Por ejemplo, obtenemos una medida que el servicio está consumiendo mucha memoria, sin embargo si tenemos una métrica que hay muchos usuarios conectados, el comportamiento es normal. Por lo tanto es aconsejable acompañar métricas personalizadas que nos permitan observar valores del sistema y de lógica de negocio. 


## Instalar Actuator y Micrometer.
En una aplicación de Spring Boot tenemos que incluir las siguientes dependencias en el archivo _pom.xml_ de Maven. 

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
            <scope>runtime</scope>
        </dependency>
```

Para exponer las métricas utilizamos el servicio de Actuator. Lo configuramos en el archivo _application.yml_ 


```yaml
management:
  endpoints:
    web:
      exposure:
        include: info,health,metrics,prometheus
```


Podemos acceder a los valores en la siguiente url:

[http://localhost:8080/actuator/prometheus](http://localhost:8080/actuator/prometheus)


## Crear un contador

Una medición muy básica es llevar la cuenta de un acontecimiento. Por ejemplo para medir la cantidad de altas de usuarios, la cantidad de emails enviados, cantidad de errores, etc. Para ello creamos un contador. 

Lo primero que necesitamos hacer es inyectar un objeto de tipo _MeterRegistry_. Este objeto tiene un método _counter_ para registrar un contador. Además invocamos al método increment para iniciar la cuenta o incrementar en el valor en una unidad.

En el siguiente ejemplo simulamos el lanzamiento de una moneda y registramos las veces que sale cara o cruz.


```java
@RestController
public class LanzarMonedaController {

    private final LanzarMoneda lanzarMoneda;
    private final MeterRegistry meterRegistry;

    public LanzarMonedaController(LanzarMoneda lanzarMoneda, 
                                  MeterRegistry meterRegistry) {
        this.lanzarMoneda = lanzarMoneda;
        this.meterRegistry = meterRegistry;
    }

    @GetMapping(value = "/moneda")
    public Moneda siguienteMoneda() {
        return obtenerMoneda();
    }

    private Moneda obtenerMoneda() {
        Moneda moneda = lanzarMoneda.siguiente();
        if (moneda == Moneda.CARA) {
            meterRegistry.counter("moneda.cara").increment();
        } else {
            meterRegistry.counter("moneda.cruz").increment();
        }
        return moneda;
    }

}
```


Podemos observar cómo se exponen las métricas si ingresamos en la url de Prometheus y buscamos los valores con los siguientes nombres:


```txt
moneda_cruz_total
moneda_cara_total
```


Todos los contadores comienzan con el nombre que le hemos indicado y terminan con total y muestra el valor acumulado.


## Medir tiempos de ejecución

Otra métrica muy útil es medir los tiempos de ejecución de una tarea, sobre todo para identificar aquellas que tardan demasiado. Por ejemplo podemos medir una consulta a la base de datos, la invocación a un servicio externo, completar un cálculo, procesar una imagen, etc. 

Creamos un temporizador utilizando el objeto _meterRegistry_ y método _timer_, pasando como parámetro un nombre para identificarlo. Luego utilizamos el método _record_ para registrar el tiempo de ejecución. Este método acepta un objeto de tipo _Runnable_. Esto hay que tenerlo en cuenta en el caso de que la tarea que necesitamos medir necesita retornar un valor. 

En el siguiente ejemplo utilizamos un _timer_ para medir el tiempo de respuesta de un método que retorna un mensaje. Como debe retornar un valor, utilizamos un objeto _AtomicReference_ para recuperarlo. 


```java
    private String obtenerMensaje() {
        AtomicReference<String> mensaje = new AtomicReference<>();
        meterRegistry
              .timer("mensaje.lento")
              .record(() -> mensaje.set(ecoLento.repite("Hola")));
        return mensaje.get();
    }
```


Las métricas que genera son las siguientes:


```txt
mensaje_lento_seconds_max
mensaje_lento_seconds_count
mensaje_lento_seconds_sum
```


Registra el tiempo máximo, la cantidad de veces que se ha invocado y el tiempo total, en segundos. 


## Medidas Gauge

Esta medición toma valores instantáneos de una magnitud que incrementa o disminuye su valor. Por ejemplo la cantidad de usuarios conectados en un chat, la cantidad de datos en un buffer, la temperatura, etc. 

Esta medida se toma siempre que sea invocada. Al contrario del contador o el timer que se miden cuando se ejecuta una acción, esta métrica se evalúa cuando se consulta su valor. 

Para crear una medida Gauge se puede utilizar un _bean_ de configuración de Spring Boot. En el ejemplo se toma un valor de un número entero aleatorio para representar una medida.


```java
@Configuration
public class GaugeConfig {

    private final ObtenerNumeroAleatorio numeroAleatorio;
    private final MeterRegistry meterRegistry;

    public GaugeConfig(ObtenerNumeroAleatorio numeroAleatorio, 
                       MeterRegistry meterRegistry) {
        this.numeroAleatorio = numeroAleatorio;
        this.meterRegistry = meterRegistry;
    }

    @Bean
    public Gauge randomGauge() {
        return Gauge
                 .builder("numero.random", numeroAleatorio, 
                          ObtenerNumeroAleatorio::siguienteEntero)
                 .register(meterRegistry);

    }
    
}
```


Las valores se muestran simplemente utilizando el nombre asignado.


```txt
numero_random 
```



## Distribución

A veces quizás no sea suficiente obtener un valor unitario de una medida, si no que necesitamos obtener mayor estadística. Entonces podemos obtener una distribución de medidas como un histograma donde cada valor obtenido se acumula en un segmento. Por ejemplo en un servicio de imágenes podemos segmentar el registro por su tamaño.

En este caso estamos configurando un objeto de tipo _DistributionSummary_ para agrupar archivos por diferentes tamaños. Tendremos la cuenta de archivos de 10 Kb o menos, de 50 o menos y de 90 como máximo. Notar que el segmento de 90 también incluye el de 50 y el de 10.


```java
@Configuration
public class DistributionSummaryConfig {

  @Bean("archivosSummary")
  public DistributionSummary getSummary(MeterRegistry meterRegistry) {
      return DistributionSummary
             .builder("archivos.sizes")
             .description("Muestra de tamaños de archivos subidos")
             .baseUnit("kilobytes")
             .serviceLevelObjectives(10, 50, 90)
             .register(meterRegistry);
  }
    
}
```


Una vez configurada, se utiliza esta métrica registrando el tamaño de cada archivo. 


```java
@RestController
public class SubirArchivoController {

    private final ObtenerNumeroAleatorio obtenerNumeroAleatorio;
    private final DistributionSummary distributionSummary;

    public SubirArchivoController(
            ObtenerNumeroAleatorio obtenerNumeroAleatorio,
            @Qualifier("archivosSummary") 
            DistributionSummary distributionSummary) {
        this.obtenerNumeroAleatorio = obtenerNumeroAleatorio;
        this.distributionSummary = distributionSummary;
    }

    @GetMapping("/archivos")
    public String subirArchivo() {
        return String.format("Se ha subido un archivo de %d Kb", 
                             getSize());
    }

    public Integer getSize() {
        Integer size = obtenerNumeroAleatorio.siguienteEntero();
        distributionSummary.record(size);        
        return size;
    }

}
```

Las métricas se muestran con el nombre asignado y la unidad de medida. Los valores que  representan son: el valor máximo, la cantidad de muestras, la suma total y la distribución en los segmentos que hemos configurado. 

```txt
archivos_sizes_kilobytes_max 
archivos_sizes_kilobytes_bucket{le="10.0",} 
archivos_sizes_kilobytes_bucket{le="50.0",} 
archivos_sizes_kilobytes_bucket{le="90.0",} 
archivos_sizes_kilobytes_bucket{le="+Inf",} 
archivos_sizes_kilobytes_count 
archivos_sizes_kilobytes_sum 
```

## Medir tiempos con anotaciones

La asignación de métricas de tiempo se pueden simplificar si utilizamos anotaciones. Se pueden marcar los métodos que necesitamos medir con la anotación _@Timed_

Para que funcione previamente necesitamos realizar algunas configuraciones. Primeramente es necesario incluir una dependencia de Spring AOP. 


```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
```


Creamos un bean de configuración para obtener una instancia de _TimedAspect_. También activamos una característica de Spring con la anotación _@EnableAspectJAutoProxy._


```java
@Configuration
@EnableAspectJAutoProxy
public class TimedConfig {

    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return  new TimedAspect(registry);
    }
    
}
```


Finalmente podemos utilizar la anotación _@Timed_ para registrar los tiempos en cada método que necesitemos medir. Notar que el método tiene que ser público. 


```java
@Repository
public class ImagenRepositoryMemory implements ImagenRepository {

    private final Random random;

    public ImagenRepositoryMemory() {
        random = new Random();
    }

    @Timed("imagen.convertir")
    @Override
    public boolean convertirImagen() {
        try {
            Thread.sleep(random.nextInt(5000));

        } catch (Exception e) {
            return false;
        }
        return true;
    }
    
}
```



## Ejemplo

Dejamos una aplicación de ejemplo en [Github](https://github.com/Mercantilandina/spring-boot-micrometer).


## Referencias
* [Documentación de Micrometer](https://micrometer.io/docs/concepts)