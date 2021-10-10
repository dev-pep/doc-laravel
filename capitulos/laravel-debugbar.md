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

El *namespace* base del paquete es ***Barryvdh\Debugbar***, y se corresponde con el directorio ***vendor/barryvdh/laravel-debugbar/src***.

## Configuración

La barra se activa automáticamente. Por un lado, se muestra solo cuando la aplicación está en modo depuración, es decir, cuando ***APP_DEBUG*** es ***true*** en el archivo ***.env***. Si no deseamos que aparezca la barra, hay que cambiar este valor a ***false***.

Por otro lado, la publicación del paquete genera el archivo ***config/debugbar.php***, en el que puede controlarse la habilitación o inhabilitación de la barra a través de la clave ***enabled***, que por defecto es ***null***. Si embargo podemos sobrescribir la activación de la barra mediante la variable ***DEBUGBAR_ENABLED*** del archivo ***.env*** estableciéndola en ***true*** o ***false***.

Por otro lado, el paquete define automáticamente un *alias*: el nombre ***Debugbar*** (en el espacio global) se refiere a la *facade* ***Barryvdh\Debugbar\Facade***.

## Uso

Así, podemos usar la *facade* que proporciona esta biblioteca de forma muy simple:

```php
\Debugbar::error('Mensaje de error...');
```

Esta sentencia inserta un mensaje de error en el *log* de errores de la barra. La *facade* tiene un método para cada nivel de *log*: `debug()`, `info()`, `notice()`, `warning()`, `error()`, `critical()`, `alert()` y `emergency()`.

Para habilitar o deshabilitar la barra en *runtime*:

```php
\Debugbar::enable();
\Debugbar::disable();
```
