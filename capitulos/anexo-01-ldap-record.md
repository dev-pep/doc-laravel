# Paquete LdapRecord-laravel

La biblioteca *LdapRecord* proporciona un modo de autenticarse en ***Directorio Activo***. Para funcionar, es necesario que *PHP* tenga instalada la extensión ***ldap***.

## Instalación

Dentro del proyecto, instalar el paquete ***ldaprecord-laravel*** con sus dependencias:

```
composer require directorytree/ldaprecord-laravel
```

Seguidamente, generaremos los archivos de configuración necesarios para su funcionamiento, es decir, publicaremos el paquete:

```
php artisan vendor:publish --provider="LdapRecord\Laravel\LdapServiceProvider"
```

## Configuración

Una vez hecho esto, se generará simplemente el archivo ***config/ldap.php***. Este archivo simplemente retorna un *array* asociativo con información sobre la conexión a *DA*.

En la clave ***default*** del *array*, indicamos el nombre de la conexión por defecto.

Otro de los elementos es ***logging***, que es un simple booleano que indica si se generará *logging* de las acciones que realicemos en el directorio *LDAP*. Los *logs* se almacenarán en ***storage/logs***.

En cuanto a las conexiones, las definimos en la clave ***connections***. Su valor será a su vez un *array* con todas las conexiones posibles a *DA*. Cada una de estas conexiones tiene una clave con el nombre de la conexión, y un valor que consiste en un *array* asociativo con los datos de la misma. Sus pares clave/valor son:

- ***hosts*** es un *array* con *strings* que definen direcciones donde encontramos servidores *LDAP*. La conexión se intentará siempre en el orden especificado.
- ***base_dn*** almacena el *distinguished name* (*DN*, ver más adelante) base, a partir de donde se realizarán todas las búsquedas y creaciones.
- ***username*** y ***password*** son las credenciales de un usuario del directorio que tenga suficientes privilegios para realizar la acción que deseamos llevar a cabo. El nombre de usuario se indicará, normalmente, con un *distinguished name*.
- ***port*** es el puerto de conexión. Si no se indica, por defecto es el 389 (conexión no *SSL*) o 636 (conexión *SSL*). Si indicamos puerto 389 pero se habilita la conexión *SSL*, el puerto queda automáticamente cambiado a 636. Si se habilita *TLS*, no se puede utilizar *SSL*, y el puerto debe ser 389.
- ***use_ssl*** y ***use_tls*** se utilizan para habilitar *SSL* o *TLS*. Solo uno de ellos puede ser ***true***. Por defecto se usa *TLS* (recomendado).
- ***timeout*** es el valor, en segundos, del *timeout*. Por defecto es 5.
- ***version*** es la versión del servidor *LDAP*. Por defecto, 3 (recomendado).
- ***follow_referrals*** es un booleano (por defecto ***false***) que indica si un servidor puede hacer un *referral* si no tiene la información pero otro de los servidores la tiene.

Los elementos ***hosts***, ***base_dn***, ***username*** y ***password*** son los únicos obligatorios.

La extensión ***ldap*** de *PHP* tiene una serie de configuraciones que podemos cambiar. Para ello utiliza las correspondientes constantes para acceder a estos valores. Si queremos cambiar alguna de estas opciones, se hace con el elemento de configuración ***options***, el cual es un *array* en el que las claves son la constante deseada. Es importante recalcar que hay algunas constantes que no se pueden redefinir a través de ***options***:

- ***LDAP_OPT_PROTOCOL_VERSION*** se cambia con el elemento ***version*** (número).
- ***LDAP_OPT_NETWORK_TIMEOUT*** se cambia con ***timeout*** (número).
- ***LDAP_OPT_REFERRALS*** se cambia con ***follow_referrals*** (booleano).


Todos estos elementos podemos definirlos en el archivo de configuración. Sin embargo, si lo dejamos tal cual está por defecto, podemos simplemente indicar la configuración en nuestro archivo ***.env*** de este modo (adaptándolo a nuestro servidor):

```
LDAP_LOGGING=true
LDAP_CONNECTION=default
LDAP_HOST=127.0.0.1
LDAP_USERNAME="cn=user,dc=local,dc=com"
LDAP_PASSWORD=secret
LDAP_PORT=389
LDAP_BASE_DN="dc=local,dc=com"
LDAP_TIMEOUT=5
LDAP_SSL=false
LDAP_TLS=false
```

Para comprobar que la conexión con el servidor *LDAP* es correcta, podemos ejecutar:

```
php artisan ldap:test
```

### *Distinguished names*

El *distinguished name* (único) para cada usuario, representa e identifica un objeto dentro de un directorio *LDAP* jerárquico (árbol). Especifica pares del tipo atributo=valor (cada uno es un *relative distinguished name*, o *RDN*), separados por comas. El árbol se recorre desde la raíz hasta el nodo, siguiendo estos pares de derecha a izquierda. Así, el primero de estos pares suele ser un nombre personal.

Los atributos posibles (*case insensitive*) son ***cn*** (nombre común), ***dc*** (componente del dominio), ***ou*** (nombre unidad de la organización), ***o*** (nombre de la organización), ***street*** (dirección postal), ***l*** (localidad), ***st*** (estado o provincia), ***c*** (país), ***uid*** (número de identidad del usuario).

En cuanto a los valores de cada uno, son los nombres que identifican las carpetas y nodos finales del directorio. Los valores de los atributos deben ser alfanuméricos *ASCII*, por lo que si deseamos indicar un nombre de dominio, incluiremos cada una de sus componentes, en atributos ***dc*** consecutivos, sin escribir los puntos.

Así, ***CN=Manolo Perez,OU=ventas,O=teshla,DC=teshlainc,DC=com*** definiría al elemento Manolo Perez. Tirando árbol arriba, este nodo pertenece al departamento (carpeta) 'ventas', de la organización 'teshla', y finalmente perteneciente al dominio de *DA* ***teshlainc.com***.

Si el directorio *LDAP* no fuese de *DA*, el atributo de más alto nivel no suele ser del tipo ***dc*** sino ***c***, por ejemplo.

Así, podríamos solicitar un elemento a un servidor *LDAP* de este modo:

```
LDAP://everest.himalaya.net:390/cn=George Mallory,ou=Alpinist,dc=himalaya,dc=net
```

## Uso

Una vez instalado, ya tendremos disponibles unos cuantos modelos *builtin* para conectarnos a un servidor de Directorio Activo o *OpenLDAP*. Existen formas de crear nuestros propios modelos *LDAP* con `artisan`, pero con los modelos incorporados es más que suficiente para la mayoría de casos.

Estos modelos podemos encontrarlos en ***vendor/directorytree/ldaprecord/src/Models*** (corresponde al *namespace* ***Ldaprecord\\Models***).

Hay, básicamente dos modos de uso: autenticación llana, y autenticación sincronizada con una base de datos. En la primera, la autenticación depende completamente del servidor *LDAP* (si ese servidor cae, nuestra aplicación no funciona). En el segundo caso, a parte del servidor, tenemos nuestra propia base de datos con la que cualquier autenticación en el *LDAP* se sincroniza. Es posible en este caso guardar información extra sobre los usuarios *LDAP*, e incluso tener usuarios que no están en el *LDAP*. Es algo complejo, y con la autenticación llana será suficiente para nuestros objetivos.

## Autenticación llana

Para configurar la conexión con el *LDAP*, hay que ir al archivo ***config/auth.php*** y buscar el *array* ***providers***, y añadirle este proveedor:

```
'users' => [
        'driver' => 'ldap',
        'model' => LdapRecord\Models\ActiveDirectory\User::class,
        'rules' => [],
    ]
```

El elemento ***rules*** define una serie de *LDAP authentication rules* (ver más abajo). Estas reglas se aplican después de que el usuario se haya autenticado con éxito en el directorio *LDAP*.

Ahora, al intentar autenticarnos, ya lo hará directamente contra el servidor de **DA** (mediante `attempt()`, de la forma habitual).

### Creación de reglas

Para crear una regla de autenticación *LDAP*:

```
php artisan make:ldap-rule NombreRegla
```

La regla se crea en ***app/Ldap/Rules***. Esta clase tiene un método `isValid()` que debe retornar ***true*** o ***false*** dependiendo de si el usuario concreto pasa la regla o no. El usuario concreto se hace disponible a través de la propiedad ***$user*** de la clase.

Una vez definida esta clase, ya podemos añadir el nombre de esta clase al *array* ***rules*** (usando `::class`, por ejemplo).
