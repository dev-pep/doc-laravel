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

O lo que es lo mismo:

```
composer create-project --prefer-dist laravel/laravel nombreProyecto ^7.0
```

Esto creará un proyecto *Laravel* en una carpeta ***nombreProyecto***, y usando la versión más actualizada de *Laravel* 7 (descartando la versión 8).

El argumento `--prefer-dist` sirve para que descargue los archivos del repositorio con la versión para distribución. Si queremos la versión del código fuente (para desarrollar, en este caso, *Laravel*) se incluiría `--prefer-source`.

Si queremos que no instale las dependencias de desarrollo, hay que incluir `--no-dev`.

Para iniciar un servidor *Laravel*:

```
php artisan serve
```

Creará un servidor web en la carpeta actual para el host ***ht<span>tp://localhost:8000***.

El archivo ***index.php*** contiene el controlador del *frontend* que gestiona todas las peticiones que llegan a la aplicación.

## Configuración

Todos los archivos de configuración están en el directorio ***config***.

Por otro lado, el archivo ***.env*** guarda los valores de entorno de la aplicación. Debería estar fuera de control de versiones, para que cada desarrollador tenga su contenido adaptado a sus necesidades. Sin embargo, el contenido que debiera ir a producción sí podría incluirse, por ejemplo en un archivo ***.env.example***.

En todo caso, si no disponemos de un archivo ***.env*** (porque acabamos de clonar un proyecto, por ejemplo), debemos generar una *key* para la aplicación (que se guarda en el archivo mencionado):

```
php artisan key:generate
```

Es importante recalcar que las dependencias del proyecto estarán en la carpeta ***vendor***, que típicamente queda fuera del control de versiones. Cuando *composer* instala nuevas bibliotecas en ese directorio, es posible que estas, para funcionar correctamente, necesiten archivos de configuración, migraciones, etc. en directorios de nuestro proyecto, fuera de ***vendor***. Estos archivos sí estarían incluidos en nuestro control de versiones.

Para que se generen tales archivos, hay que ejecutar:

```
php artisan vendor:publish
```

En ese momento, las bibliotecas que no lo hayan hecho, contribuirán con sus archivos de configuración. Normalmente, crearán un archivo *.php* dentro del directorio ***Config***, aunque pueden crear otras cosas, como se ha dicho.

### Variables de entorno

Las variables definidas en ***.env*** son *overriden* por las variables existentes del sistema.

Las variables establecedas en ***.env*** se leen como *strings*, con lo que se pueden usar valores como ***true***, ***false***, ***null***, o ***empty*** (que se interpretará como un *string* vacío).

Si a una variable hay que darle un valor que contiene espacios, se indicará entre comillas dobles.

Todos los valores de este archivo (así como las variables de entorno del sistema) se cargarán en la superglobal de *PHP* `$_ENV`, aunque se puede usar la función `env()` para acceder a estas variables:

```php
$d = env('APP_DEBUG', false);
```

El segundo parámetro (opcional) es el valor por defecto que retornará la función si no encuentra tal variable de entorno.

### Acceso a los valores de configuración

Los valores de configuración se almacenan en el directorio ***config***. Allí se encuentran los archivos de configuración. Estos son simples archivos *PHP* que lo que hacen es retornar un *array* de pares clave-valor.

Para acceder a los valores de configuración definidos, se usa la función `config()`. Para acceder a un valor concreto, se usa la sintaxis de punto. La función tiene un segundo parámetro opcional con el valor por defecto:

```php
$C = config('app.timezone', 'Asia/Seoul');
```

En este caso, se accede a un archivo ***config/app.php*** que retorna un *array* que contiene una clave ***timezone***. La función retorna el valor asociado a esa clave. Sin embargo, esto también accedería a un archivo ***config/app/timezone.php***, retornando el valor o *array* completo que retorna este archivo. Así, la sintaxis de puntos se usa para descender tanto por directorios y subdirectorios como por *arrays* y *subarrays*. El conjunto de directorios y *arrays* forma un árbol. El *helper* `config()` no tiene por qué solicitar una hoja del árbol; si especifica un nodo no terminal (sea un *array* o un directorio), retornará un *array* con el resto de la jerarquía hasta los nodos terminales.

Es posible establecer un valor de configuración en tiempo de ejecución. Para ellos pasaremos un *array* a `config()` estableciendo los valores que deseemos:

```
config(['app.timezone' => 'Madrid/Paris']);
```

Para que la aplicación funcione más rápido (normalmente, en entorno de producción), se puede configurar la aplicación para que agrupe todos los valores de configuración en un solo archivo que será cargado al principio:

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

- ***app*** - core de la aplicación. Contiene los modelos.
    - ***app/Console*** - API a la aplicación en modo consola (comandos `artisan`).
    - ***app/Http*** - API a la aplicación (controladores, *middleware*, *requests*).
    - ***app/Providers*** - contiene los *service providers* de la aplicación.
- ***bootstrap*** - contiene ***app.php*** (instancia de la aplicación), que arranca el *framework*.
- ***config*** - archivos de configuración (autodocumentados).
- ***database*** - contiene migraciones de base de datos.
- ***public*** - contiene el ***index.php*** con la configuración de *autoloading*, y punto de entrada de todas las peticiones a la aplicación. También almacena los *assets*: imágenes, *javascript*, *css*,...
- ***resources*** - contiene las **vistas** y archivos de idiomas.
- ***routes*** - definición de todas las rutas de la aplicación.
- ***storage*** - plantillas *Blade* compiladas, caches, y otros archivos generados por la aplicación. La subcarpeta ***storage/logs*** guarda los *logs* generados, ***storage/framework*** los archivos que necesite generar el propio *framework*, y ***storage/app*** los archivos que genere la aplicación.
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
