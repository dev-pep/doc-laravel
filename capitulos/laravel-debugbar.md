# Laravel-DebugBar

Esta librería permite obtener estado y mensajes de depuración en una barra.

La librería utiliza a su vez la librería *PHP DebugBar*.

## Instalación

Desde el directorio de nuestro proyecto, usaremos `composer`:

```
composer require barryvdh/laravel-debugbar --dev
```

El paquete es relevante solo en las instalaciones en desarrollo.

Tras haber hecho esto, el paquete es autodescubierto por *Laravel*, con lo que no es necesario añadir el *service provider* manualmente.

Seguidamente, generaremos los archivos de configuración necesarios para su funcionamiento, es decir, publicaremos el paquete:

```
php artisan vendor:publish --provider="Barryvdh\Debugbar\ServiceProvider"
```

El *namespace* base del paquete es ***Barryvdh\\Debugbar***, y se corresponde con el directorio ***vendor/barryvdh/laravel-debugbar/src***.

## Configuración

La barra se activa automáticamente. Por un lado, se muestra solo cuando la aplicación está en modo depuración, es decir, cuando ***APP_DEBUG*** es ***true*** en el archivo ***.env***. Si no deseamos que aparezca la barra, hay que cambiar este valor a ***false***.

Por otro lado, la publicación del paquete genera el archivo ***config/debugbar.php***, en el que puede controlarse la habilitación o inhabilitación de la barra a través de la clave ***enabled***, que por defecto es ***null***. Si embargo podemos sobrescribir la activación de la barra mediante la variable ***DEBUGBAR_ENABLED*** del archivo ***.env*** estableciéndola en ***true*** o ***false***.

Por otro lado, el paquete define automáticamente un *alias*: el nombre ***Debugbar*** (en el espacio global) se refiere a la *facade* ***Barryvdh\\Debugbar\\Facade***.

## Uso

Así, podemos usar la *facade* que proporciona esta biblioteca de forma muy simple:

```php
\Debugbar::error('Mensaje de error...');
```

Esta sentencia inserta un mensaje de error en la pestaña de mensajes de la barra, a nivel de error. La *facade* tiene un método para cada nivel de *log*: `debug()`, `info()`, `notice()`, `warning()`, `error()`, `critical()`, `alert()` y `emergency()`. El método `addMessage()` añade el mensaje al nivel por defecto (por defecto `info()`).

### *Collectors*

La barra dispone de una serie de *collectors* integrados, cada uno de los cuales genera una pestaña o un indicador (normalmente estos están en la parte derecha). Es posible habilitar y inhabilitar los *collectors* que deseemos, en la clave ***collectors*** del archivo de configuración de la barra.

Los métodos que hemos visto insertan los mensajes en la pestaña ***Messages***.

Cuando tal mensaje no es un simple valor (como un *string*), sino un *array* o un objeto complejo, la salida será similar a un volcado (*dump*) de variable del tipo `dump()` de *Laravel*. El *helper* `debug()` realiza una salida de nivel *debug*, pasándole todas las expresiones que deseemos ver.

En la pestaña ***Timeline*** se pueden cronometrar fragmentos de código. Una de las formas de hacerlo es:

```php
\Debugbar::startMeasure('nombre', 'Descripción de la medición');
// Código a cronometrar
\Debugbar::stopMeasure('nombre');
```

Si queremos cronometrar una función explícitamente:

```php
\Debugbar::measure('nombre_operacion', function {
    // Código de la función a cronometrar
});
```

Estos métodos disponen de su *helper* asociado: `start_measure()`, `stop_measure()` y `measure()`.

### Excepciones

Si disponemos de un objeto excepción, por ejemplo al capturar tal excepción, podemos enviar tal excepción a la barra:

```php
try {
    // código
}
catch(Exception $e) {
    \Debugbar::addThrowable($e);
}
```

### Activación/desactivación

Para habilitar o deshabilitar la barra en *runtime*:

```php
\Debugbar::enable();
\Debugbar::disable();
```
