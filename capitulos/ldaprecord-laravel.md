# LdapRecord-Laravel

Esta librería permite trabajar con servidores *LDAP*. Para funcionar, es necesario asegurarse de que se cumplen los requisitos (versión *PHP*, extensión *ldap*).

La librería utiliza a su vez la librería *PHP* *LdapRecord*.

## Instalación

Desde el directorio de nuestro proyecto, usaremos `composer`:

```
composer require directorytree/ldaprecord-laravel
```

Seguidamente, generaremos los archivos de configuración necesarios para su funcionamiento, es decir, publicaremos el paquete:

```
php artisan vendor:publish
    --provider="LdapRecord\Laravel\LdapServiceProvider"
```

El *namespace* base del paquete es ***LdapRecord\\Laravel***, y se corresponde con el directorio ***vendor/directorytree/ldaprecord-laravel/src***.

## Configuración

Una vez publicado el paquete, se generará simplemente el archivo ***config/ldap.php***. Este archivo define las conexiones a los servidores *LDAP*. Estas conexiones se integran en el contenedor de conexiones de *LdapRecord*.

- En la clave ***default*** del *array*, indicamos el nombre de la conexión por defecto.
- La clave ***connections*** es un *array* con las conexiones a diferentes servidores *LDAP*. Cada conexión tiene como clave el nombre de la misma, y como valor, un *array* con los datos de la conexión: ***hosts***, ***base_dn***, ***username***, ***password***, ***port***, ***use_ssl***, ***use_tls***, ***timeout***, ***version***, ***follow_referrals*** y ***options***. Para más información, véase el paquete *LdapRecord*.
- Otro elemento es ***logging***, que es un simple booleano que indica si se generará *logging* de las acciones que realicemos en el directorio *LDAP*. Los *logs* se almacenarán en ***storage/logs***.

El archivo de configuración puede tener los valores *hardcoded*, o hacer referencia a variables de entorno (o archivo ***.env***):

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

## Uso

Una vez instalado el paquete, disponemos de los modelos que proporciona el paquete *LdapRecord*, situados en ***vendor/directorytree/ldaprecord/src/Models*** (corresponde al *namespace* de ese paquete, es decir, ***Ldaprecord\\Models***).

Si deseamos crear nuestros propios modelos, podemos hacerlo con `artisan`:

```
php artisan make:ldap-model NombreModelo
```

Esto creará un modelo en ***app/Ldap*** (crea la carpeta si no existe). El modelo creado extiende, por defecto, el modelo ***Ldaprecord\\Models\\Model***. Pero se puede cambiar a cualquier otro. Por ejemplo, si queremos funcionalidades específicas de Directorio Activo, podemo extender ***Ldaprecord\\Models\\ActiveDirectory\\Entry***.

Dado que estos modelos están definidos en el paquete *Ldaprecord*, véase la documentación de dicho paquete para más información.

## Autenticación básica

Un método simple de autenticación es utilizando un objeto conexión (véase paquete *Ldaprecord*):

```php
use LdapRecord\Container;
use LdapRecord\Models\ActiveDirectory\User;

$conexion = Container::getConnection('default');
$usuario = User::find('samaccountname', 'pepe');
if($conexion->auth()->attempt($usuario->getDn(), 'secreto'))
{
    // autenticación ok
}
```

### *Guards*, proveedores, conexiones y modelos

Para dar una visión general comprensible: cada *guard* (definidos en ***config/auth.php***) representa un método de autenticación. Los principales atributos de un *guard* son el *driver* (sesión o token) y el proveedor de usuarios.

Cada proveedor de usuarios (***config/auth.php***) tiene dos atributos importantes: el *driver* (normalmente ***eloquent*** o ***database***, pero en este caso será ***ldap***) y el modelo. En el caso de un proveedor *ldap* tenemos otros atributos, como ***rules***.

En el caso del modelo, este está asociado a una conexión del contenedor de *LdapRecord*. Si no tiene conexión definida, usará la *default*.

## Autenticación multidominio

Es posible especificar más de una conexión en ***config/ldap.php***, es decir, especificar varios servidores *LDAP*. Para ello debemos usar varios modelos, cada uno de los cuales usará una de las conexiones definidas. Para ello, cada uno de estos modelos tendrá una propiedad ***\$connection***, que contendrá el nombre de la conexión deseada.

Para cada una de estas conexiones hay que especificar, lógicamente, un *guard* (en ***config/auth.php***), y un proveedor asociado a ese *guard*. Cada uno de esos proveedores tendrá un modelo asociado que indicará qué conexión usará (y restricciones opcionales a aplicar).

## Restricciones de acceso (*rules*)

Para restringir acceso según pertenencia a grupos, se usan las reglas, que se aplican tras la autenticación. Para crear una regla:

```
php artisan make:ldap-rule NombreRegla
```

En el ejemplo anterior, se creará una regla en ***app/Ldap/Rules/NombreRegla.php***.

La regla tiene un método `isValid()` que retornará ***true*** si el usuario pertenece a los grupos adecuados, y ***false*** en caso contrario. Una vez definida la regla, se añadirá al valor de la clave ***rules*** del *array* correspondiente a la definición del *guard*.

La clase dispone de dos atributos:
- ***\$user*** es una instancia del modelo *LdapRecord* correspondiente al usuario autenticado.
- ***\$model*** es una instancia del modelo *Eloquent*, en el caso de autenticación sobre una base de datos local sincronizada con el servidor *LDAP*.

```php
public function isValid()
{
    return $this->user->groups()
        ->exists('Finanzas', 'Marketing');
}
```

El ejemplo retornará ***true*** si el usuario tiene **todos** los grupos indicados en los argumentos. Si deseamos que se compruebe si el usuario pertenece a **uno o más** de los grupos indicados, usaremos el método `contains()`.

En todos los casos, los grupos indicados pueden ser *common names*, *distinguished names* o instancias del modelo ***LdapRecord\\Models\\ActiveDirectory\\Group***.

```php
public function isValid()
{
    return $this->user->groups()->exists(
        Group::find(
            'cn=Finanzas,ou=Groups,dc=tesla,dc=com'),
        Group::find(
            'cn=Marketing,ou=Groups,dc=tesla,dc=com'));
}
```

Si la regla retorna ***false***, el usuario no quedará autenticado.

## Autenticación llana

Cuando un usuario se autentica correctamente, ***Auth::user()*** retorna una instancia de nuestro modelo *LdapRecord*.

Para autenticar de forma llana:

```php
Auth::attempt([
    'mail' => 'pepe@pepe.com',
    'password' => 'secreto'
]);
```

El método retorna ***true*** si la autenticación tiene éxito. En ese caso, como de costumbre se tiene acceso al usuario mediante ***Auth::user()*** (instancia del modelo), se puede comprobar la autenticación con ***Auth::check()***, etc.

## Autenticación con base de datos sincronizada

Es posible sincronizar el directorio *LDAP* con una base de datos local. En ese caso, todos los usuarios que se autentiquen con éxito contra el directorio *LDAP* serán creados y sincronizados en una tabla local. A partir de entonces, toda autenticación de ese usuario se hará contra la tabla local. A la base de datos local se accede a través de un modelo *Eloquent*.

La tabla de usuarios de la base de datos local puede tener los campos que deseemos, tanto si coinciden con campos del directorio *LDAP*, como si son campos con informaciones extras que nos interesa guardar. Sin embargo, la tabla deberá tener obligatoriamente el campo ***guid*** para almacenar el *UID* del usuario en el directorio, y el campo ***domain***, con el nombre de la conexión *LDAP* del usuario.

La migración encargada de añadir esos campos está disponible para publicación:

```
php artisan vendor:publish
    --provider="LdapRecord\Laravel\LdapAuthServiceProvider"
php artisan migrate
```

El siguiente paso es añadir el *trait* ***LdapRecord\\Laravel\\Auth\\AuthenticatesWithLdap*** y la interfaz ***LdapRecord\\Laravel\\Auth\\LdapAuthenticatable*** al modelo. Esto permite la lectura y escritura de los campos ***guid*** y ***domain*** en las autenticaciones.

Finalmente, se pueden cambiar los nombres de esos dos campos en la migración publicada, pero en ese caso hay que *override* los métodos `getLdapDomainColumn()` y/o `getLdapGuidColumn()` en nuestro modelo:

```php
use LdapRecord\Models\Model;
use LdapRecord\Laravel\Auth\LdapAuthenticatable;
use LdapRecord\Laravel\Auth\AuthenticatesWithLdap;

class User extends Model implements LdapAuthenticatable
{
    use AuthenticatesWithLdap;

    // ...

    public function getLdapDomainColumn() {
        return 'my_domain_column';
    }

    public function getLdapGuidColumn() {
        return 'my_guid_column';
    }
}
```

### Configuración

Una vez hecho esto, hay que configurar este proveedor de usuarios adecuadamente (en ***config/auth.php***) para que funcione esta sincronización. Concretamente, hay que añadir la clave ***database*** con la información necesaria. Esta clave contendrá un *array* valores útiles para la sincronización:

- ***model***: modelo *Eloquent* de la tabla local.
- ***sync_passwords***: booleano que indica si el *password* también debe almacenarse en la tabla local.
- ***sync_attributes***: *array* con correspondencias de nombres entre la tabla local (claves) y el directorio *LDAP* (valores). La columna de la contraseña no hace falta indicarla, si tiene en ambos lados el nombre ***password***, o si no se quiere sincronizar.

```php
'providers' => [
    'users' => [
        'driver' => 'ldap',
        'model' =>  // modelo ldaprecord
            LdapRecord\Models\ActiveDirectory\User::class,
        'rules' => [],
        'database' => [
            'model' => App\User::class,  // modelo eloquent
            'sync_passwords' => false,
            'sync_attributes' => [
                'name' => 'cn',
                'email' => 'mail',
            ],
        ],
    ],
],...
```

La autenticación se realiza con la *facade* ***Auth*** de la forma habitual.

## Autenticación *SSO*

La autenticación *Single Sign-On* (llamada también *Windows authentication*) permite al usuario autenticarse una sola vez en el servidor, de tal modo que no es necesario que se autentique en cada aplicación. Para ello, **el servidor** ***web*** solicita autenticación para acceder a una *URL*, y si tiene éxito, almacena el nombre del usuario autenticado en una variable de entorno. En *IIS* esa variable es ***USER_AUTH***, mientras que en servidores *Apache* es ***REMOTE_USER***. Si la autenticación se hace sobre un directorio *LDAP* (normalmente será *AD*), el valor almacenado en esta variable de entorno es el campo ***sAMAccountName***.

Desde *PHP* es posible leer esa variable con `$_SERVER['REMOTE_USER']`. En *Laravel* puede hacerse con `env('REMOTE_USER')`. *LdapRecord-Laravel* dispone de un *middleware* que comprueba el contenido de esta variable, y si localiza el usuario en el directorio lo autentica automáticamente.

### *Middleware*

El *middleware* que realiza esta tarea es ***LdapRecord\\Laravel\\Middleware\\WindowsAuthenticate***. Habría que añadirlo en el grupo de *middleware* que corresponda a nuestras necesidades. Este *middleware* utiliza también las *rules* que hayamos definido en ***config/auth.php***.

Para cambiar el nombre de la variable de entorno del servidor que almacena el nombre de usuario, se debe ejecutar el método estático del *middleware* `serverKey()`, pasándole el nuevo nombre. En nuestro caso haríamos:

```php
WindowsAuthenticate::serverKey('REMOTE_USER');  // servidor
                                                // Apache
```

Un buen lugar para hacerlo es en el método `boot()` de un *service provider*, como ***AuthServiceProvider***.

En un servidor *Apache*, si un usuario no perteneciente al dominio entra en la aplicación, la variable ***REMOTE_USER*** retornará ***null***, y el *middleware* será ignorado (*bypassed*), permitiendo que a estos usuarios autenticarse manualmente. De esta forma habrá usuarios autenticados con *SSO* y otros autenticados de otra forma.
