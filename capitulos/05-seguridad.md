# Seguridad

## Autenticación

El mecanismo de autenticación que proporciona *Laravel* pasa por los *guards* (definen cómo se autentica el usuario) y los proveedores (bases de datos de usuarios).

>*Laravel* permite la creación *out of the box* de todo el sistema de autenticación mediante una serie de *starter kits*. Para una descripción detallada de estos, véase la documentación oficial.

La **configuración** de la autenticación en *Laravel* se halla en el archivo ***config/auth.php***.

Este archivo retorna un *array* que tiene varios elementos, cuyas claves son ***defaults***, ***guards***, ***providers***, ***passwords*** y ***password_timeout***.

- La clave ***providers*** describe los proveedores, es decir, las fuentes persistentes de las que se extraen los datos de usuario. Su valor es un *array* cuyos elementos son dichos proveedores: cada proveedor tiene una clave con el nombre que daremos a ese proveedor, y su valor es, a su vez, un *array*, con estos elementos:
    - El elemento con clave ***driver*** podrá tener el valor ***eloquent*** si el proveedor es un modelo *Eloquent*, o ***database*** si el proveedor es simplemente una tabla, sin modelo asociado.
    - El elemento ***model*** almacenará el nombre de la clase del modelo si el proveedor es un modelo. En caso contrario, tendrá un elemento ***table*** con el nombre de la tabla.
- En ***guards*** definimos el guardián (método) de autenticación. Su valor es un *array* cuyos elementos son dichos *guards*: cada uno de ellos tendrá una clave con el nombre que daremos a ese *guard*, y su valor es un *array*, con estos elementos:
    - El elemento ***driver*** indica el método de autenticación. Actualmente solo admite el valor ***session*** (para *requests web*). Para autenticación de *requests API* es necesario crear código para gestionarla, o instalar paquetes adicionales que se encarguen de ello.
    - El elemento ***provider*** es el nombre de uno de los proveedores que hemos definido.
- El elemento ***defaults*** indica, en su clave ***guard***, cuál es el *guard* por defecto.

Por defecto, *Laravel* proporciona un modelo *Eloquent* ***App\\Models\\User***, válido para acceder a la tabla de usuarios, y su correspondiente migración (tabla ***users***). Si queremos definir los campos de la tabla de usuarios nosotros mismos, hay que tener en cuenta que debe existir un campo ***password*** de mínimo 60 caracteres de anchura (si usamos `string()` en el esquema de la migración sin especificar anchura será un ***VARCHAR*** *MySQL* de 255 caracteres, lo cual ya está bien). Por otro lado, si queremos disponer de la característica *remember me*, la tabla debe contener un campo llamado ***remember_token*** de 100 caracteres.

### Mecanismo

La autenticación funciona almacenando las credenciales del usuario en la sesión de este (las sesiones se guardan en el servidor). Por otro lado, se envía una *cookie* con el *ID* de la sesión al navegador, con lo que en cada *request* se enviará dicha *ID* para que el servidor pueda saber de qué sesión se trata, y asociar así la sesión al usuario correcto. Si la sesión incluye credenciales válidas, se considerará al usuario como autenticado.

Por otro lado, una *request API* no dispone de tal mecanismo, ya que estas peticiones no están asociadas a la navegación de un usuario (puede ser un servicio remoto intentando obtener datos, una petición asíncrona, etc.). Para autenticarse, las peticiones *API* envían un *API token* en cada petición. Ese *token* se compara con la lista de *tokens* válidos en la base de datos, de tal modo que así puede saberse qué usuario (o grupo de usuarios) está asociado a tal *token*. Dado que este *token* se suele enviar en la *query string*, en necesario renovarlo con frecuencia.

En *Laravel* se accede a los mecanismos de autenticación a través de las *facades* ***Illuminate\\Support\\Facades\\Auth*** y ***Illuminate\\Support\\Facades\\Session***, para conseguir autenticación basada en sesión (navegadores *web*).

### Autenticación de usuarios

Para autenticar a un usuario, se usa el método `attempt()` (intento de autenticación) de la *facade* ***Auth***, al que se le pasa un *array* con campos de la tabla de usuarios, para ver si tal usuario se encuentra en ella. Como mínimo, debería estar el campo que usemos para autenticarnos (nombre de usuario, normalmente) y la contraseña. Cuando esta está en el campo ***password***, *Laravel* le aplica automáticamente un *hash* y la compara con la de la tabla (que está *hashed*). Si todo coincide, se inicia automáticamente una sesión autenticada. Este método, además, retorna ***true*** si la autenticación es correcta, o ***false*** en caso contrario.

```php
public function authenticate(Request $request)
{
    $credentials = $request->only('username', 'password');
    if(Auth::attempt($credentials)) {
         $request->session()->regenerate();  // regeneramos la sesión
         return redirect('dashboard');  // autenticación OK
    }
    return redirect('inicio');  // autenticación fallida
}
```

***Auth*** usará el *guard* por defecto, accediendo al modelo o tabla para buscar la información del usuario.

La regeneración de la sesión cambia la *session ID*, y es útil para evitar el ataque de *session fixation*.

> **Session fixation:** el *hacker* envía un enlace a la víctima para que entre en https://elbanco.com?sessionid=23498jhwefpi8ufasd9p8yu. Como vemos, el *hacker* conoce el campo donde va la *session ID*, con lo que se inventa un *ID* y lo añade en la *query string*. La víctima hace clic en el enlace. Como esta sesión es nueva, le pide sus credenciales del banco. El usuario se loguea tranquilamente. Una vez logueado, el *hacker* puede acceder al sitio usando esa *session ID*, y estará logueado sin necesidad de conocer las credenciales.
>
> Por otro lado, si cuando la víctima se loguea la *session ID* se regenera, entonces ese *ID* que tiene el *hacker* no le sirve para nada.

Por otro lado, el método `attempt()` acepta más campos, a parte del usuario y el *password*:

```php
if(Auth::attempt(['username' => $usname, 'password' => $password, 'active' => 1]))
    // ...
```

Se iniciará la sesión autenticada y retornará ***true*** solo si coinciden todos los campos (en este caso, también comprueba que el usuario esté marcado como activo).

Si queremos que la autenticación se realice mediante un *guard* que no sea el *default*, usaremos el método `guard()` antes de la llamada a `attempt()`:

```php
if(Auth::guard('un_guard')->attempt($credentials))
    // ...
```

Para saber si existe usuario autenticado, lo haremos mediante `Auth::check()`, que retorna ***true*** si el usuario está autenticado.

Existen dos formas de acceder al usuario autenticado actualmente: mediante `Auth::user()`, o a través del método `user()` de la *request*. El usuario obtenido es el modelo *Eloquent* o registro de la tabla pertinente.

Por otro lado, para obtener el *ID* del usuario autenticado, se puede hacer con `Auth::id()`.

### *Middleware* de autenticación

*Laravel* tiene un *middleware* cuyo nombre es ***auth***, que puede aplicarse a todas las rutas que necesiten usuario autenticado para acceder. En tal caso, si accedemos a una ruta de estas sin usuario autenticado, se realizará automáticamente una redirección a la ruta cuyo nombre sea ***login***.

Simplemente se añadirá el método `middleware('auth')` a la cadena de métodos aplicados a la ruta (o grupo de rutas). Por otro lado, si deseamos que se aplique un *guard* concreto a estas rutas, se añadirá `middleware('auth:admin')` (en este caso, el *guard* ***admin***).

### Recordar al usuario

Si queremos autenticar a un usuario activando la opción *remember me*, pasaremos a `attempt()` como segundo argumento (opcional) un valor ***true***. De este modo, el usuario permanece logueado permanentemente (hasta que hace *logout* explícitamente). Este mecanismo utiliza el campo ***remember_token*** de la tabla o modelo de usuarios.

Es posible saber si un usuario logueado lo ha hecho a través de un *login* corriente o a través de un *remember me*, usando el método `Auth::viaRemember()`, que retorna ***true*** en el último caso.

### Otras formas de autenticación

Es posible autenticar a un usuario concreto. Simplemente se debe obtener el registro deseado de la tabla o modelo, y pasarlo al método `login()` (no es necesario *password*):

```php
$user = User::where('username', 'manolito')->first();
Auth::login($user);
```

También se puede autenticar a un usuario por su clave primaria:

```php
Auth::loginUsingId(476378);
```

Estos dos métodos funcionan de forma similar a `attempt()` en cuanto a especificar el *guard* deseado o activar *remember me*.

Por otro lado se puede autenticar a un usuario del mismo modo que si se tratase de `attempt()` pero solo para la *request* actual, de tal modo que no se generarían *cookies*, ni sesiones, ni nada. Resulta útil para una petición *API*, por ejemplo. Para ello simplemente se usaría el método `once()`.

### Autenticación *HTTP* básica

El *middleware* (incluido por defecto) ***auth.basic*** proporciona un mecanismo de entrada de credenciales sin una página de *login* específica. Por defecto, este *middleware* asume como nombre de usuario el campo ***email*** de la tabla de usuarios.

En algunos casos, si el servidor es *Apache*, podría ser necesario añadir esta configuración en el servidor:

```
RewriteCond %{HTTP:Authorization} ^(.+)$
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

Es decir, si se trata de la cabecera de autenticación, se creará una variable del servidor con el contenido de dicha cabecera.

Por otro lado, podríamos crear un *middleware* que realizara esta autenticación básica para peticiones *API* (autenticación *stateless*). Para ello, se haría la autenticación mediante `Auth::onceBasic()`. Si retornara ***true***, el *middleware* seguiría para adelante.

Recordemos que luego habría que registrar este *middleware* (propiedad ***\$routeMiddleware*** de ***app/Http/Kernel.php***) y asignarlo a la ruta (o grupo de rutas) con el método `middleware()`.

De todas formas, solicitar credenciales para peticiones *API* no tiene por qué ser una buena idea normalmente.

### *Log out*

Para desautenticar al usuario, simplemente hay que ejecutar `Auth::logout()`. Esto elimina la información de autenticación de la sesión.

Es aconsejable tras el *logout* invalidar la sesión actual y regenerar el *token CSRF*:

```php
public function logout(Request $request)
{
    Auth::logout();
    $request->session()->invalidate();
    $request->session()->regenerateToken();
    return redirect('/');
}
```




> En cuanto a la caducidad de la sesión, depende del navegador usado. En algunos sistemas y navegadores, la sesión termina al cerrar el navegador, mientras en otros no es así. No debemos asumir que una sesión se ha cerrado si no lo hemos hecho explícitamente.










## Autorización

Se utilizan dos mecanismos para gestionar la autorización de un usuario a utilizar recursos. Son las *gates* (puertas) y *policies* (políticas).

### *Gates*

Las puertas son *closures* (o métodos) que definen si un usuario está autorizado a realizar cierta acción. Se registran en un *service provider*. Por conveniencia, se puede hacer en la clase ***App\\Providers\\AuthServiceProvider*** que viene por defecto en *Laravel*. Un *gate* recibe por defecto una instancia del usuario, y otros argumentos opcionales.

En el método `boot()` del proveedor de servicios registraremos la puerta mediante el método `define()` de la *facade* ***Gate***:

```php
Gate::define('update-post', function ($user, $post) {
        return $user->id === $post->author_id;
    });
```

El primer argumento a `define()` es el nombre que damos a esa puerta. El segundo, es una *closure* que retornará ***true*** o ***false*** según se permita la acción o no. En nuestro caso, definimos una puerta relacionada con la acción de actualizar un *post* en un blog. La acción se permitirá si y solo si el usuario autenticado es el autor del *post*. Para ello, vemos que la *closure* recibe dos argumentos: en primer lugar, una instancia del usuario autenticado (que se le pasa automáticamente). Después de este primer parámetro, vienen los otros argumentos que deseemos darle (en este caso, una instancia del *post*, que incluye información del autor).

En lugar de una *closure* podemos indicar un *string* del tipo 'Clase@método', igual que con los controladores.

```php
Gate::define('update-post', 'App\Policies\PostPolicy@update');
```

Una vez hemos registrado la puerta, ya podemos utilizarla desde cualquier parte del código:

```php
if(Gate::allows('update-post', $post))
    { /* Autorizado */ }
if(Gate::denies('update-post', $post))
    { /* NO autorizado */ }
```

Como vemos, tanto `allows()` como su contrario `denies()` toman, como primer argumento, el nombre de la puerta. Si al definir la puerta hemos indicado un parámetro adicional (en este caso ***$post***), estos métodos lo recibirán como segundo argumento. Sin embargo, si hemos definido más de uno, los recibirán en un *array*.

Se puede comprobar una autorización, no solo para el usuario autenticado, sino para cualquiera de ellos:

```php
if(Gate::forUser($usuario)->allows('update-post', $post))
    { /* ,,, */ }
if(Gate::forUser($usuario)->denies('update-post', $post))
    { /* ... */ }
```

Si deseamos saber si un usuario tiene autorización para pasar por lo menos una de las puertas indicadas, usaremos el método `any()`, al que, en lugar del nombre de la puerta pasaremos un *array* con varios nombres. De forma similar, si queremos saber si el usuario no tiene autorización para pasar ninguna de las puertas indicadas, usaremos `none()`.

Existen también directivas *Blade* para comprobar si el usuario tiene autorización (`@can`, `@cannot`, `@canany`), que reciben los argumentos de forma análoga a los métodos de comprobación.

Se pueden registrar funciones que se llevarán a cabo antes y después de todas las comprobaciones de autorización (con `allows()`, etc.). La sintaxis es similar a `define()`, pero sin ese primer parámetro con el nombre de la puerta (se ejecutan siempre). Así, su primer parámetro es la instancia del usuario autenticado.

Por un lado, el método `before()` definirá las acciones a realizar antes de cualquier chequeo. Si retorna un valor no nulo, ese será el resultado final de las comprobaciones. Si retorna ***null***, el resultado de la comprobación dependerá de las funciones sucesivas.

En cuanto a `after()`, se ejecuta después de las comprobaciones habituales. Si retorna un valor no nulo, *overrides* el valor de la comprobación.

### *Policies*

Las políticas organizan el acceso a recursos o modelos concretos. Si por ejemplo tenemos un modelo ***Coche***, podríamos tener una política ***CochePolicy***. La podríamos crear así:

```
php artisan make:policy CochePolicy
```

Creará una clase vacía para esa política en ***app/Policies/CochePolicy.php***. Si queremos que incluya lógica para un *CRUD* estándar, podemos indicarle el modelo al que estará asociada:

```
php artisan make:policy CochePolicy --model=Coche
```

Las políticas son resueltas por el *sevice container*, con lo que pueden ser inyectadas automáticamente en nuestras clases.

Para registrar las políticas, podemos hacerlo en ***App\\Providers\\AuthServiceProvider***, donde disponemos ya de la propiedad ***policies***. Esta es un *array* que asocia cada modelo a la política de acceso a él:

```php
protected $policies = [
        Coche::class => CochePolicy::class
    ];
```

De todas formas, *Laravel* autodescubre las políticas existentes, siempre que se cumpla:

- La política debe estar asociada a un modelo concreto de tal modo que el nombre de la política sea el mismo que el del modelo, con el sufijo ***Policy***.
- La política debe estar en una carpeta ***Policies*** dentro de la carpeta donde reside el modelo.

Si lo deseamos podemos definir nuestras propias formas de autodescubrimiento. Eso se realiza registrando estas formas así:

```php
Gate::guessPolicyNamesUsing(function ($modelClass) {
    // retornar el nombre de la clase de la política
});
```

Esto debería hacerse, típicamente, dentro del método `boot()` de ***AuthServiceProvider***. En este caso, ***$modelClass*** es el nombre de la clase del modelo.

Por otro lado, las asociaciones realizadas en la propiedad ***$policies*** del *service provider* tienen preferencia sobre cualquier tipo de autodescubrimiento.

Al desarrollar una política, debemos escribir métodos en ella, para cada una de las acciones que deseemos controlar. Estos métodos recibirán normalmente un primer argumento con el usuario autenticado, y un segundo con una instancia del modelo. El método retornará ***true*** si se autoriza el acceso, y ***false*** en caso contrario.

Las acciones de la política que podemos describir son, por ejemplo, `view()`, `create()`, `update()`, `delete()` o `restore()`.

En algunos casos, como `create()`, el método solo recibirá un argumento: el usuario autenticado.

Una vez definida la política, ya podemos usarla. Existen varios modos.

Una manera es a través del modelo ***User*** que viene incluido en *Laravel*. En este caso, tras obtener una instancia del usuario, esta dispone de los métodos `can()` y `cant()`, a los que se pasará, en primer lugar, el nombre de la acción a comprobar; y en segundo lugar, una instancia del modelo sobre el que el usuario desea ejecutar esa acción:

```php
if($user->can('update', $coche))
    { /* ... */ }
```

En el caso de comprobar acciones como `create`, más que una instancia del modelo (que no existe), le pasaremos el nombre de la clase:

```php
use App\Coche;

if($user->can('create', Coche::class))
    { /* ... */ }
```

Existe otro modo de usar las políticas, y es mediante las directivas *Blade* `@can`, `@cannot` y `@canany`.

```html
@can('update', $coche)
    <!-- código html para usuarios que pueden actualizar el elemento $coche -->
@elsecan('create', App\Coche::class)
    <!-- código html para usuarios que NO pueden actualizarlo pero sí crearlo -->
@endcan
```

Sería similar para un bloque `@cannot` - `@elsecannot` (opcional) - `@endcannot`.

El ejemplo anterior equivale exactamente a:

```html
@if(Auth::user()->can('update', $coche))
    <!-- código html para usuarios que pueden actualizar el elemento $coche -->
@elseif(Auth::user()->can('create', App\Coche::class))
    <!-- código html para usuarios que NO pueden actualizar el elemento $coche -->
@endif
```

El inverso a `@can('update', $coche)` sería `@unless(Auth::user()->can('update', $coche))`.

También podemos comprobar si el usuario tiene por lo menos una de las capacidades listadas en un array mediante la directiva `@canany`:

```html
@canany(['update', 'view', 'delete'], $coche)
    <!-- para usuarios que pueden actualizar, ver o borrar el elemento $coche -->
@elsecanany(['create'], \App\Coche::class)
    <!-- para usuarios que no pueden hacer lo anterior pero sí crear un coche -->
@endcanany
```

Como hemos visto en los ejemplos, en acciones que no precisan de instancia del modelo (como `create`), se pasa el nombre de la clase.

## Encriptación

Toda la encriptación en *Laravel* se realiza internamente a través de la *facade* ***Crypt***. *Laravel* la utiliza para encriptar *cookies* (para asegurarse de que no se modifican en el lado del cliente), etc. Para ello, esta *facade* necesita una clave, configurada en ***config/app.php***, y que por defecto toma su valor de la variable ***APP_KEY*** en el archivo ***.env***.

Al crear un nuevo proyecto con *Composer*, o mediante el instalador de *Laravel*, se genera una clave automáticamente, pero al clonar una aplicación, puesto que el archivo ***.env*** no suele formar parte del repositorio, no se obtiene una clave por defecto. Además, aunque la clave también se clonara, no sería buena idea utilizarla: no es buena idea que nuestra aplicación use una clave que ya usa otra aplicación. Nuestra clave debería ser única y existir solo en nuestra aplicación. Por lo tanto, cuando **clonamos** un proyecto deberemos crear nuestra propia clave para la aplicación:

```
php artisan key:generate
```  

Al ejecutar este comando, se cambia la clave por una aleatoria, y si no existe el archivo ***.env*** se crea uno automáticamente.

Hay que indicar que los *passwords* en *Laravel* no son encriptados, sino *hashed* (mediante el método `make()` de la *facade* ***Hash***, o con el *helper* equivalente `bcrypt()`), con lo que no usan la clave de la aplicación, lo que significa que generar una clave nueva no afectará a los *passwords* almacenados. En este caso, el *hash* es *one-way*, mientras que la encriptación es de ida y vuelta, y **simétrica** (la misma clave de la aplicación se puede aplicar para encriptar y para desencriptar), ya que usa *AES-256-CBC*.

Si deseamos rotar la clave de la aplicación, hay que tener en cuenta que al cambiarla, las *cookies* encriptadas existentes serán invalidadas, y cualquier dato almacenado que hayamos encriptado con ***Crypt*** ya no será desencriptable. Si nuestra aplicación es servida desde varios servidores, hay que configurar la misma clave en todos ellos.

Para encriptar un *string* se usa `Crypt::encryptString()`; se le pasa el *string* a encriptar y retorna en *string* encriptado. Exactamente lo contrario que `Crypt::decryptString()`.
