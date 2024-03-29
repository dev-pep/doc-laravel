# Primeros pasos

*Laravel* es un *framework PHP*. Es una estructura de funcionalidades creadas en *PHP* que automatiza, es decir, simplifica enormemente, el trabajo con aspectos muy frecuentemente utilizados, como el tratamiento de la solicitud (*request*) entrante, la respuesta (*response*) retornada al cliente, las cabeceras *HTTP* entrantes y salientes, la autenticación del usuario, el trabajo con bases de datos, el manejo de la sesión o la seguridad, entre otras cosas.

Por lo tanto, resulta imprescindible conocer el lenguaje *PHP* para poder utilizar *Laravel* de forma eficiente.

## Instalación

*Laravel* se puede instalar, tras instalar sus dependencias, a través del instalador del mismo *Laravel*, o a través de *Composer*. En ambos casos necesitamos *Composer*, que es la forma que tiene *Laravel* de gestionar las dependencias.

Para instalar el propio instalador:

```
composer global require laravel/installer
```

Hay que asegurarse que la carpeta de ejecutables de los paquetes instalados globalmente en *Composer* está en el *path* del sistema. Esta carpeta está en la carpeta ***vendor/bin*** dentro del directorio de configuración de *Composer* (que suele variar de un sistema a otro).

Una vez instalado, para crear un proyecto, se ejecuta, desde la carpeta en la que deseamos que se cree la subcarpeta del proyecto:

```
laravel new nombreProyecto
```

Por otro lado, para instalar un proyecto *Laravel* directamente:

```
composer create-project --prefer-dist laravel/laravel
    nombreProyecto
```

O lo que es lo mismo:

```
composer create-project --prefer-dist laravel/laravel
    nombreProyecto
```

Si queremos una versión concreta de *Laravel*, cualquiera de estas dos órdenes vale:

```
composer create-project --prefer-dist laravel/laravel:^7.0
    nombreProyecto

composer create-project --prefer-dist laravel/laravel
    nombreProyecto ^7.0
```

Esto creará un proyecto *Laravel* en una carpeta ***nombreProyecto***, usando la versión más actualizada de *Laravel*.

El argumento `--prefer-dist` sirve para que descargue los archivos del repositorio con la versión para distribución. Si queremos la versión del código fuente (para desarrollar, en este caso, *Laravel*) se incluiría `--prefer-source`.

Si queremos que no instale las dependencias de desarrollo, hay que incluir `--no-dev`.

Para iniciar un servidor *Laravel*:

```
php artisan serve
```

Creará un servidor web en la carpeta actual para el host ***ht<span>tp://localhost:8000***.

El archivo ***index.php*** es el *front controller*, que gestiona todas las peticiones que llegan a la aplicación.

## Configuración

Todos los archivos de configuración están en el directorio ***config***.

Por otro lado, el archivo ***.env*** guarda los valores de entorno de la aplicación. Debería estar fuera de control de versiones, para que cada desarrollador tenga su contenido adaptado a sus necesidades. Sin embargo, el contenido que debiera ir a producción sí podría incluirse, por ejemplo en un archivo ***.env.example***. Este no debería incluir una *key* de aplicación, para que se generara una nueva para este despliegue.

> La variable ***APP_URL*** del archivo ***.env*** no juega un papel importante en el funcionamiento de la aplicación. Sin embargo, algunos paquetes la usan y no funcionarán bien si está mal configurada. A parte de esto, algunos componentes, como las notificaciones por correo electrónico o algunos comandos de consola, también se pueden ver afectados.

En todo caso, si no disponemos de un archivo ***.env*** (porque acabamos de clonar un proyecto, por ejemplo), debemos generar una *key* para la aplicación (que se guarda en el archivo mencionado):

```
php artisan key:generate
```

Es importante recalcar que las dependencias (bibliotecas) del proyecto estarán en la carpeta ***vendor***, que típicamente queda fuera del control de versiones. Cuando *composer* instala nuevas bibliotecas en ese directorio, es posible que estas, para funcionar correctamente, necesiten archivos de configuración, migraciones, etc. en directorios de nuestro proyecto, fuera de ***vendor***. Estos archivos sí estarían incluidos en nuestro control de versiones.

Para que se generen tales archivos, hay que ejecutar:

```
php artisan vendor:publish
```

En ese momento, las bibliotecas que no lo hayan hecho, contribuirán con sus archivos de configuración. Normalmente, crearán un archivo *.php* dentro del directorio ***config***, aunque pueden crear y/o modificar otros otros archivos, como se ha dicho.

### Variables de entorno

El *helper* `env()` tiene acceso a las variables de entorno del sistema, así como a los valores de las superglobales *PHP* ***\$\_ENV*** y ***\$\_SERVER***. En caso de no existir la variable de entorno o la variable superglobal, se buscará el valor en el archivo ***.env***.

```php
$d = env('APP_DEBUG', false);
```

El segundo parámetro (opcional) es el valor por defecto que retornará la función si no encuentra tal variable de entorno (o superglobal), y tampoco un valor en el archivo ***.env***.

Las variables del sistema y superglobales tienen, pues, prioridad sobre las definiciones en ***.env***. Si se llega a utilizar un valor en este archivo, se incluye dentro del *array* ***\$\_ENV***.

Las variables establecedas en ***.env*** se leen como *strings*, y se pueden usar valores como ***true***, ***false***, ***null***, o ***empty*** (que se interpretará como un *string* vacío).

Si a una variable hay que darle un valor que contiene espacios, se indicará entre comillas dobles.

### Acceso a los valores de configuración

Los valores de configuración se almacenan en el directorio ***config***. Allí se encuentran los archivos de configuración. Estos son simples archivos *PHP* que lo que hacen es retornar un *array* de pares clave-valor.

Algunos valores de configuración pueden referirse a valores en ***.env*** (configuración dependiente del entorno).

Para acceder a los valores de configuración definidos, se usa la función `config()`. Para acceder a un valor concreto, se usa la sintaxis de punto. La función tiene un segundo parámetro opcional con el valor por defecto:

```php
$C = config('app.timezone', 'Asia/Seoul');
```

En este caso, se accede a un archivo ***config/app.php*** que retorna un *array* que contiene una clave ***timezone***. La función retorna el valor asociado a esa clave. Sin embargo, esto también accedería a un archivo ***config/app/timezone.php***, retornando el valor o *array* completo que retorna este archivo. Así, la sintaxis de puntos se usa para descender tanto por directorios y subdirectorios como por *arrays* y *subarrays*. El conjunto de directorios y *arrays* forma un árbol. El *helper* `config()` no tiene por qué solicitar una hoja del árbol; si especifica un nodo no terminal (sea un *array* o un directorio), retornará un *array* con el resto de la jerarquía hasta los nodos terminales.

Es posible establecer un valor de configuración en tiempo de ejecución. Para ellos pasaremos un *array* a `config()` estableciendo los valores que deseemos:

```php
config(['app.timezone' => 'Madrid/Paris']);
```

Para que la aplicación funcione más rápido (normalmente, en entorno de producción), se puede configurar la aplicación para que agrupe todos los valores de configuración en un solo archivo cache que será cargado al principio:

```
php artisan config:cache
```

> Si hacemos esto, las llamadas a `env()`, fuera de los archivos de configuración, retornarán siempre ***null***.

Para eliminar ese archivo cache de configuración:

```
php artisan config:clear
```

Si se ha creado un archivo cache de configuración y posteriormente se modifica dicha configuración (directorio ***config*** y/o archivo ***.env***), habrá que eliminar o regenerar dicho cache.

### Modo de depuración

Por defecto, la configuración de depuración (en ***config/app.php***) se determina por la opción ***debug***. Esta variable puede ser ***true*** o ***false*** según deseemos el modo de depuración. Dicha opción suele basarse en el valor de la variable de entorno (en ***.env***) ***APP_DEBUG***.

Cuando el modo de depuración está activo, se muestra mucha información sobre un posible error encontrado en el código. Esta información no se muestra con el modo desactivado, aunque sigue siendo accesible a través del *log* de errores (siempre que esté debidamente configurado, es decir, activo y con nivel de *logging* ***debug***).

### Modo de mantenimiento

Ejemplos:

```
php artisan down
php artisan down --redirect=/mantenimiento
php artisan up
```

## Estructura de directorios

La estructura puede organizarse a voluntad. Veremos aquí parte de la estructura por defecto. Directorios:

- ***app*** - core de la aplicación. Contiene los modelos.
    - ***app/Console*** - API a la aplicación en modo consola (comandos `artisan`).
    - ***app/Http*** - API a la aplicación (controladores, *middleware*, *requests*).
    - ***app/Providers*** - contiene los *service providers* de la aplicación.
- ***bootstrap*** - contiene ***app.php*** (instancia de la aplicación), que arranca el *framework*; también contiene el directorio ***cache***.
- ***config*** - archivos de configuración (autodocumentados).
- ***database*** - contiene migraciones de base de datos y otras utilidades.
- ***lang*** - contiene los archivos de idiomas.
- ***public*** - contiene el ***index.php*** con la configuración de *autoloading*, y punto de entrada de todas las peticiones a la aplicación. También almacena los *assets*: imágenes, *javascript*, *css*,...
- ***resources*** - contiene las **vistas** y otros recursos.
- ***routes*** - definición de todas las rutas de la aplicación.
- ***storage*** -
    - ***storage/app*** - archivos generados por la aplicación.
    - ***storage/logs*** - *logs* generados.
    - ***storage/framework*** - archivos generados por el propio *framework*.
- ***tests*** - *tests* automatizados.
- ***vendor*** - dependencias de *Composer*.

### El directorio *app*

En cuanto al directorio *app* (correspondiente al *namespace* ***App***), tiene estas subcarpetas por defecto:

- ***Console*** - comandos *artisan*.
- ***Exceptions*** - manejador de excepciones. Es un buen lugar para almacenar las excepciones creadas a medida.
- ***Http*** - contiene toda la lógica de tratamiento de una *request* (controladores, *middleware*).
- ***Models*** - contiene las clases de los modelos *ORM* (*object-relational model*) *Eloquent*.
- ***Providers*** - *service providers* de la aplicación.

Existen otras subcarpetas (***Broadcasting***, ***Events***, ***Jobs***, ***Listeners***, ***Mail***, ***Notifications***, ***Policies***, ***Rules***), que almacenan otros mecanismos del *framework*, y que existen solo en caso de usar tales mecanismos (se irán viendo más adelante).

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

Esto crea un archivo cache de rutas que acelera el registro de las mismas. Para eliminar tal archivo:

```
php artisan route:clear
```

Por otro lado:

```
php artisan view:cache
```

Precompila todos los *templates Blade*, con lo que no se hace *on demand* cada vez que se retorne una vista a partir de una petición.

Para eliminar todas las plantillas precompiladas (bajo demanda o por el comando anterior):

```
php artisan view:clear
```

Por otro lado, se debería establecer la variable ***APP_DEBUG*** a ***false***, para no exponer información sobre valores de configuración.

En cuanto a la variable ***APP_ENV***, es aconsejable indicar ***production*** en lugar de ***local*** (para desarrollo). Esta variable es útil cuando nuestra aplicación se comporta de forma diferente en producción que en desarrollo, o incluso en otras fases del ciclo de vida que se nos ocurran. Si en el código no hacemos distinción alguna, su valor es indiferente.
