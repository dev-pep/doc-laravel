# Primeros pasos

## Instalación

*Laravel* se puede instalar, tras instalar sus dependencias, a través del instalador del mismo *Laravel*, o a través de *Composer*. En ambos casos necesitamos *Composer*, que es la forma que tiene *Laravel* de gestionar las dependencias.

Para instalar el propio instalador:

```
composer global require laravel/installer
```

Hay que asegurarse que la carpeta de ejecutables de los paquetes instalados globalmente en *Composer* está en el *path* del sistema. Esta carpeta está en la carpeta ***vendor/bin*** dentro del directorio de configuración de *Composer* (que suele variar de un sistema a otro).

Para instalar un proyecto *Laravel* directamente:

```
composer create-project --prefer-dist laravel/laravel:^7.0 nombreProyecto
```

Esto creará un proyecto *Laravel* en una carpeta ***nombreProyecto***, y usando la versión más actualizada de *Laravel* 7 (descartando la versión 8).

Para iniciar un servidor *Laravel*:

```
php artisan serve
```

Creará un servidor web en la carpeta actual para el host ***ht<span>tp://localhost:8000***.

El archivo ***index.php*** contiene el controlador del *frontend* que gestiona todas las peticiones que llegan a la aplicación.

## Configuración

Todos los archivos de configuración está en el directorio ***config***. El archivo ***.env*** guarda los valores de entorno de la aplicación. Debería estar fuera de control de versiones, para que cada desarrollador tenga su contenido adaptado a sus necesidades. Sin embargo, el contenido que debiera ir a producción sí podría incluirse, por ejemplo en un archivo ***.env.example***.

### Variables de entorno

Las variables definidas en ***.env*** son *overriden* por las variables existentes del sistema.

Las variables establecedas en ***.env*** se leen como *strings*, con lo que se pueden usar valores como ***true***, ***false***, ***null***, o ***empty*** (que se interpretará como un *string* vacío).

Si a una variable hay que darle un valor que contiene espacios, se indicará entre comillas dobles.

Todos los valores de este archivo se cargarán en la superglobal de *PHP* `$_ENV`, aunque se puede usar la función `env()`:

```php
$d = env('APP_DEBUG', false);
```

El segundo parámetro (opcional) es el valor por defecto que retornará la función si no encuentra tal variable de entorno.

### Acceso a los valores de configuración

Para acceder a los valores de configuración definidos, se usa la función `config()`. También tiene un segundo parámetro opcional con el valor por defecto:

```php
$C = config('app.timezone', 'Asia/Seoul');
```

Para que la aplicación funcione más rápido (normalmente, en entorno de producción), se puede configurar la aplicación para que agrupe todas los valore de configuración en un solo archivo que será cargado al principio:

```
php artisan config:cache
```

> Si hacemos esto, las llamadas a `env()` fuera de los archivos de configuración retornarán siempre ***null***.

### Modo de mantenimiento

Ejemplos:

```
php artisan down
php artisan down --message="Actualizando base de datos"
php artisan down --allow=127.0.0.1 --allow=192.168.0.0/16

php artisan up
```

## Estructura de directorios

La estructura puede organizarse a voluntad. Veremos aquí parte de la estructura por defecto. Directorios:

- ***app*** - core de la aplicación.
  - ***app/Console*** - API a la aplicación en modo consola (comandos `artisan`).
  - ***app/Http*** - API a la aplicación (controladores, *middleware*, *requests*).
  - ***app/Providers*** - contiene los *service providers* de la aplicación.  
- ***bootstrap*** - contiene ***app.php***, que arranca el *framework*.
- ***config*** - archivos de configuración (autodocumentados).
- ***database*** - contiene migraciones de base de datos.
- ***public*** - contiene el ***index.php*** con la configuración de *autoloading*, y punto de entrada de todas las peticiones a la aplicación. También almacena los *assets*: imágenes, *javascript*, *css*,...
- ***resources*** - contiene las **vistas** y archivos de idiomas.
- ***routes*** - definición de todas las rutas de la aplicación.
- ***storage*** - plantillas *Blade* compiladas, caches, y otros archivos generados por la aplicación. La subcarpete ***storage/logs*** guarda los *logs* generados, ***storage/framework*** los archivos que necesite generar el propio *framework*, y ***storage/app*** los archivos que genere la aplicación.
- ***tests*** - *tests* automatizados.
- ***vendor*** - dependencias de *Composer*.

## Despliegue

Algunas notas sobre el *deployment* a producción de aplicaciones *Laravel*.

El comando:

```
composer install --optimize-autoloader --no-dev
```

optimiza el mapa de *autoload* de *Composer*, lo cual acelera la autocarga.

Se recomienda usar el comando `php artisan config:cache` explicado anteriormente.

Si hay muchas (cientos) rutas, se recomienda:

```
php artisan route:cache
```

Esto acelera el registro de rutas.

```
php artisan view:cache
```

Esto precompila todos los *templates Blade*, con lo que no se hace *on demand* cada vez que se retorne una vista a partir de una petición.
