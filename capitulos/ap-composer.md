# Composer

*Composer* es un gestor de paquetes *PHP*. Puede utilizarse en cualquier proyecto realizado en lenguaje *PHP*, aunque es un compañero inseparable de *frameworks* como *Laravel*.

## Instalación

Se instala ejecutando el *script* que nos proporciona <https://getcomposer.org/>. Para descargarlo:

```
curl https://getcomposer.org/installer -o composer-setup.php
```

Seguidamente, se ejecuta `php composer-setup.php`. El *script* deja en el directorio actual un archivo ***composer.phar***, que se ejecuta como:

```
php composer.phar
```

En este momento, se puede eliminar el *script* de instalación (si no se ha hecho ya automáticamente).

En cuanto al nuevo *script* generado, se puede ejecutar directamente como `composer.phar` en sistemas *Unix* (sin necesidad de invocar `php`), ya que se crea con permisos de ejecución, y posee tiene la correspondiente línea *shebang*.

Si queremos que esté disponible en todo el sistema, se debe copiar a un directorio en el ***PATH***, como por ejemplo, ***/usr/local/bin***. De lo contrario habrá que especificar la ruta del *script* (`./composer.phar` si está en el directorio actual).

También es útil renombrarlo como ***composer*** a secas.

Todo esto puede hacerse en un solo paso (en sistemas *Unix*) si en el momento de ejecutar el instalador lo hacemos de una forma similar a esta:

```
php compoer-setup.php --install-dir=/usr/local/bin
    --filename=composer
```

Los paquetes que queramos instalar a nivel global, no solo en el proyecto actual, se guardan en el directorio de configuración de *composer*. En *Linux*, los archivos ejecutables de un paquete instalado **globalmente** van al directorio ***~/.config/composer/vendor/bin***.

Por ejemplo, si queremos disponer del instalador de aplicaciones *laravel*, teclearemos:

```
composer global require laravel/installer
```

Esto instalará el instalador. Para que los ejecutables de los paquetes instalados globalmente en *composer* estén disponibles en el ***PATH***, editaremos nuestro ***.bashrc***, añadiendo:

```
export PATH="$PATH:$HOME/.config/composer/vendor/bin"
```

Ahora ya podríamos usar el instalador de *laravel* desde cualquier sitio:

```
laravel new nombreApp
```

Esto crearía un directorio ***nombreApp*** con un proyecto *laravel* (con su correspondiente archivo ***index.php*** en ***nombreApp/public***, etc.).

## Paquetes

Para ver los paquetes disponibles, se puede navegar en <https://packagist.org>. Es el repositorio por defecto.

Los nombres de los proyectos se componen de nombre del *vendor* (desarrollador del paquete) y el nombre del paquete en sí, de la forma ***vendor/paquete***.

Todas las dependencias del mismo paquete quedan reflejadas en el archivo ***composer.json***. De hecho, cualquier proyecto que contenga un archivo ***composer.json*** es así mismo un paquete.

Cuando instalamos paquetes con *composer*, se crea un directorio ***vendor*** en el directorio raíz del proyecto, que es donde instalará las dependencias del proyecto.

> Si usamos *git*, es buena idea incluir el directorio ***vendor*** en ***.gitignore***.

Al requerir un paquete específico, se buscan primero en los repositorios que hayamos definido en ***composer.json***:

```json
{
    "repositories":
    [{
        "type": "composer",
        "url": "http://packages.example.com"
    },
    {
        "type": "vcs",
        "url": "https://github.com/Seldaek/monolog"
    }]
}
```

La *URL* puede ser también un directorio local o en red.

El paquete debe tener **obligatoriamente** definido un nombre (***name***) dentro de su `composer.json`. Este debe ser un nombre completo (***vendor/paquete***).

Puede tener definido también otros campos como ***description*** o ***type*** (***library***), autor (***author***), etc.

La búsqueda de un paquete se realiza en el orden indicado. Si no se encuentra el paquete en ninguno de los repositorios, se busca en *packagist.org*. Existe otro formato para indicar los repositorios, usando sintaxis de objeto:

```json
{
    "repositories": {
         "foo": {
             "type": "composer",
             "url": "http://packages.foo.com"
         }
    }
}
```

En este caso, sin embargo, el orden de búsqueda no está garantizado.

En cuanto al sistema de versiones, en *Git* se definen mediante etiquetas (*tags*). En este caso, la nomenclatura de etiquetas acepta una letra ***v*** como prefijo opcionalmente. Por otro lado, también en el caso específico de repositorios *Git*, al tirar de la versión concreta, se descarga también la carpeta ***.git***.

> Cuando requerimos una biblioteca desde un repositorio *Git* para desarrollo de esa biblioteca, es posible que el *snapshot* no sea el adecuado. Normalmente *Composer* elige un *snapshot* asociado a la etiqueta (versión) más adecuada. Por lo tanto, tras hacer el `require` de la biblioteca, es probable que se deba cambiar a la rama de desarrollo apropiada mediante el comando `switch` de *Git*.

Existen varios tipos de repositorio (***type***):

- ***composer*** se trata de un archivo ***packages.json*** con información *dist* y *source* (ver más adelante). Admite un campo ***options*** con opciones.
- ***vcs*** repositorio de un sistema de control de versiones. Puede ser *Git* (se puede indicar ***git***), *SVN*, y otros.
- ***package*** permite indicar la fuente *dist* y *source* de un paquete que no esté integrado con el sistema de *composer* (es decir, sin ***composer.json***, dependencias, etc.).

Dentro de un repositorio puede haber dos fuentes: *source* y *dist*. El primero es el código fuente, y el segundo es el paquete empaquetado para distribución (en un archivo ***zip***). En el caso de ***vcs*** no existe fuente *dist*, con lo que se clona el repositorio en el proyecto (en la carpeta pertinente de ***vendor***) y se hace un *checkout* según las restricciones de versión que hayamos definido en ***composer.json***. Para ello, se tendrán en cuenta los mecanismos del repositorio (*tags* y ramas).

Para indicar si debemos descargar la versión *dist* o *source*, se configurará según cada paquete (o grupo de paquetes, admitiendo *wildcards*). Por ejemplo:

```json
{
    "config": {
        "preferred-install": {
            "my-organization/stable-package": "dist",
            "my-organization/*": "source",
            "partner-organization/*": "auto",
            "*": "dist"
        }
    }
}
```

***auto*** utiliza ***source*** en paquetes en desarrollo, y ***dist*** en paquetes en producción. El *matching* se hace de arriba hacia abajo, con lo que los patrones más específicos deben indicarse antes.

## Paquetes de la plataforma

En *composer* pueden definirse (a través de ***composer.json***) versiones de paquetes que no son gestionados por *composer* mismo, sino por el entorno en el que se encuentra. Se trata de paquetes (*virtuales*) instalados en el sistema sobre los que *composer* no tiene ningún control. De este modo podemos requerir una versión (o versiones) de *PHP* (***php***), una extensión de este (***ext-<nombreext>***), o una biblioteca del sistema (***lib-<nombrelib>***). La utilidad de esto es que *composer* dará un mensaje de error si no se cumplen estas dependencias.

## Comandos

El archivo `composer.json` especifica las propiedades del proyecto. En el apartado ***require*** aparecen todas las dependencias del proyecto.

```json
"require": {
    "monolog/monolog": "^1.0",
    "acme/foo": "^2.0"
}
```

A parte, se puede especificar la versión de *PHP* que necesita el proyecto, y las extensiones *PHP* que deben estar instaladas:

```json
"require": {
    "PHP": ">=7.2",
    "ext-ldap": "*",
    "ext-ftp": "*"
}
```

Normalmente no se indica restricción de versión en las extensiones *PHP*, ya que van ligadas a la versión del mismo *PHP*, que hemos definido ya.

`composer init` ejecuta un *wizard* que solicita información del proyecto. Autor, tipo, licencia,... Crea un `composer.json` nuevo con la información aportada.

`composer require thevendor/thepackage` descarga el paquete *thevendor/thepackage* en la carpeta ***vendor***, y actualiza el `composer.json` adecuadamente con ese requisito.

`composer require --dev thevendor/thepackage` hace lo mismo que el anterior, pero en lugar de añadirlo a la *key* `"require"`, lo añade a la `"require-dev"`. Estos paquetes no se instalan en un servidor de producción, solo en uno de desarrollo.

`composer update` descarga y actualiza los paquetes instalados (y los no existentes pero requeridos) a la versión más reciente posible según requisitos y restricciones. Se encarga de instalar también todas las dependencias de todos los paquetes. Si hay paquetes en `vendor` que no están requeridos en `composer.json` y no son necesarios, los elimina. Actualiza (o crea, si no existe) ***composer.lock***, que es una **instantánea** que describe los paquetes instalados, indicando las versiones que están instaladas actualmente.

Si solo deseamos actualizar un paquete determinado:

```
composer update vendor/paquete
```

`composer install` instala los paquetes según la definición de `composer.lock`, es decir, recrea esa última instantánea. Esto es útil, por ejemplo, si queremos replicar las mismas dependencias en otro equipo: solo hay que copiar `composer.lock` en el otro equipo y ejecutar `composer install` en él para tener exactamente el mismo estado de dependencias en los dos lados. En el caso de que no exista `composer.lock`, ejecuta `composer update`.

`composer remove thevendor/thepackage` desinstala (elimina) el paquete especificado de disco, y la entrada correspondiente de `composer.json`.

`composer create-project thevendor/thepackage [dir]` instala el paquete, pero en lugar de instalar en `vendor` y añadir a `composer.json`, lo descarga como proyecto a la carpeta `dir` (que si no lo especificamos, se llamará como el paquete). Esta carpeta resultante es la carpeta del nuevo proyecto.

Si tecleamos simplemente `composer create-project` en un directorio que contiene `composer.json`, instala los paquetes requeridos en el directorio actual directamente, en lugar de en `vendor` u otra carpeta. La carpeta actual es la carpeta del nuevo proyecto.

Se puede especificar una versión al crear un proyecto. Por ejemplo, para crear un proyecto *laravel 7.0*:

```
composer create-project laravel/laravel myproject 7.0
```

o

```
composer create-project laravel/laravel:7.0 myproject
```

El comando `global` se indica justo antes de `install`, `update`, `require` y `remove`, para trabajar en el directorio de configuración de *composer*, haciendo que los paquetes estén disponibles para todos los proyectos del sistema.

### Autoloading

Para las bibliotecas que proporcionan autocarga de clases, *composer* crea un archivo ***vendor/autoload.php***, de tal modo que incluyendo ese archivo en algún punto de nuestro proyecto, quedarán todas esas bibliotecas autocargadas en nuestro código fuente. Aunque podríamos incluir este archivo con `include`, es mejor hacerlo con `require`.

Si queremos añadir elementos propios al *autoload*, podemos hacerlo en el archivo ***composer.json*** **de la biblioteca**:

```json
{
    "autoload": {
        "psr-4": {"Acme\\": "src/"},
        "psr-4": {"Acme\\Fuu\\": "src2/"}
    }
}
```

Cada *namespace* base (relativo al *namespace* global) se corresponde con un directorio base (relativo al directorio raíz de la biblioteca, donde reside ***composer.json***). Subdirectorios de este directorio se corresponden con *subnamespaces* del *namespace* base. Cada archivos *PHP* en esta jerarquía se corresponde con una clase de la jerarquía de *namespaces*.

> **Importante:** los nombres de los subdirectorios deben llamarse igual que los nombres de los *subnamespaces*, y tener una **capitalización idéntica**, al igual que los nombres de los archivos y las clases que definen. Es posible que en algún sistema se permita una capitalización distinta, pero para evitar problemas es mejor ajustarse al estándar.

Un archivo de clase *PHP* situado en un punto concreto de la jerarquía debe definir un *namespace* (`namespace <nombre>`) coincidente con el *subnamespacio* que le corresponde.

Una vez esto está definido correctamente, hay que ejecutar:

```
composer dump-autoload
```

o

```
composer dumpautoload
```

De este modo, se añadirá el *autoload* de todas las clases pertinentes, en ***vendor/autoload.php***.

Cabe añadir que solo se tendrán en cuenta para el archivo de autocarga las bibliotecas que hayan sido efectivamente instaladas (requeridas) mediante *Composer*, es decir, posibles bibliotecas que se hayan copiado "a mano" no tendrán efecto (ya que si se copia simplemente no queda sincronizada con el cache de *Composer*).

#### PSR-4

El ejemplo anterior añade un *autoloader PSR-4* para el *namespace* ***Acme*** (*PSR-4* define el mapeo entre los *namespaces* y los directorios, y el nombre de la clase con el nombre del archivo que la define). En este sentido, estamos definiendo nuestro propio mapeo, del directorio ***src*** (relativo al **raíz del proyecto**) al *namespace* ***Acme***.

En el ejemplo anterior, si en el código fuente aparece la clase ***Acme/Foo/Faa/MyClase***, se buscará en ***src/Foo/Faa/MyClase.php***. Y la clase ***Acme/Fuu/Faa/MyOtraClase***, se buscará en ***src2/Faa/MyOtraClase.php***.

Los *namespaces* deben terminar en barra invertida, ya que si no, `"Acme"` coincidiría con clases en *namespaces* como ***AcmeFoo***, ***AcmeBar***,...

Es posible buscar una clase en más de un directorio hasta encontrarlo:

```json
"psr-4": {"Acme\\": ["src/", "src2/", "src3/"]}
```

También podemos definir un (o varios) directorio *fallback* para el resto de clases:

```json
"psr-4": {"": "src/"}
```

#### PSR-0

*PSR-0* es similar a *PSR-4*, pero con algunas diferencias que le dan compatibilidad con viejos estilos de mapeo. En el ejemplo anterior, en lugar de mapear ***Acme/Fuu/Faa/myOtraClase*** a ***src2/Faa/myOtraClase.php***, lo haría a ***src2/Acme/Fuu/Faa/myOtraClase.php***. Por otro lado, permite también mapear, no solo *namespaces*, sino el nombre de una clase.

#### Classmap y Files

El tipo de *autoloading* `classmap` busca en los directorios y archivos especificados y **escanea** todas las clases que estén ahí definidas (en archivos *.php* y *.inc*). En los nombres se pueden usar el *wildcard* ***\****.

Si queremos excluir archivos del *classmap*, se usará `exclude-from-classmap`. Se puede usar el *wildcard* ***\****, que coincide con todos los caracteres exceptuando la barra (***/***), o ***\*****, que coincide con todos los caracteres sin excepción.

Por otro lado, se pueden requerir (`require`) explícitamente archivos usando `files`.

```json
{
    "autoload": {
        "classmap": [
            "src/addons/*/lib/",
            "3rd-party/*",
            "Something.php"
        ],
        "exclude-from-classmap": [
            "/Tests/",
            "/test/",
            "/tests/"
        ],
        "files": ["src/MyLibrary/functions.php"]
    }
}
```

## Restricción de versiones

Las restricciones se pueden editar en el `composer.json` o especificarse con el comando `composer require`, como por ejemplo:

```
composer require "vendor/packa:2.*" "vendor/packb:~2.1"
```

### Formato de las restricciones

- `*` - la más actual posible.
- `1.5` - versión única.
- `>=1.0` indica mayor o igual a 1.0.
- `>=1.0 <2.0` indica mayor o igual a 1.0 **y** menor a 2.0. Cuando hay varias condiciones, un espacio o una coma actúa como ***AND***. Si indicamos `||`, se entenderá ***OR***.
- `>=1.0 <1.1 || >=1.2` indica mayor o igual a 1.0 y menor a 2.0, o también mayor o igual a 1.2.
- `1.0.*` equivale a `>=1.0 <1.1`.
- `1.*` equivale a `>=1.0 <2.0`.
- `1.0-2.0` equivale a `>=1.0.0 <2.1.0`. Decir `<2.1` es lo mismo que decir `<2.1.0`.
- `1.0.0-2.1.0` equivale a `>=1.0.0 <=2.1.0`.
- `~1.2` equivale a `>=1.2 <2.0`.
- `~1.2.3` equivale a `>=1.2.3 <1.3.0`.

El operador `~` permite versiones en las que **únicamente el último** número especificado puede ser tan grande como se desee:
- `^1.2` equivale a `>=1.2 <2.0`.
- `^1.2.3` equivale a `>=1.2.3 <2.0`.
- El operador `^` permite versiones hasta la *siguiente versión mayor* (no incluida), para evitar que se rompa la posible compatibilidad. Como excepción, en las versiones *pre-1.0*, se considera el segundo número como versión mayor:
- `^0.3` equivale a `>=0.3 <0.4`.

Hay otras indicaciones de tipo `dev-master`, `alpha`, etc.

### Etiquetas (*tags*) del *VCS*

Cuando *composer* examina un repositorio *Git*, se basa en las etiquetas asignadas a los *commits* para decidir qué *commit* descargará. Las etiquetas en *Git* se usan para indicar el número de versión de algunos *commits*, y suelen usar la nomenclatura ***v1.0***, ***v3.2.0***, etc. *Composer* elimina la ***v*** inicial para tomar sus decisiones.

Al buscar en el repositorio, *composer* busca **todas** las etiquetas, y elige la más alta posible según las restricciones que le hemos dado.

### Estabilidad

*Composer* reconoce estos niveles de estabilidad: en orden creciente son ***dev***, ***alpha***, ***beta***, ***RC*** y ***stable***. En el *VCS*, para decidir la estabilidad de una versión (de un *commit*), se indica en el sufijo: ***v1.3-RC1*** (estabilidad ***RC***), ***v2.1-BETA*** (estabilidad ***beta***), etc. Si no se indica tal sufijo, *composer* asume ***stable***.

Dicho esto, en lugar de especificar la restricción de versiones según un rango de números de versión, es posible indicar el mínimo grado de estabilidad deseado. En este caso, se usará ***minimum-stability***, asociándole ese valor de la estabilidad mínima deseada.

En caso de aceptar una estabilidad inferior a estable, podemos indicar con ***prefer-stable*** a ***true*** que si existe alguna versión estable, la utilice.

```json
{
    "minimum-stability": "dev",
    "prefer-stable": true
}
```
