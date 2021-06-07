# Seguridad

## Autenticación

>*Laravel* permite la creación *out of the box* de un *frontend* para la autenticación de usuarios, con sus controladores y vistas. Esto se hace, en primer lugar, instalando el paquete de *Composer* llamado ***laravel/ui***, y ejecutando después:
>
>```
>php artisan ui vue --auth
>```
>
>Hay otras opciones para el *frontend*, con lo que si en lugar de `vue` queremos presentarlo con `bootstrap` también podemos hacerlo.
>
>En todo caso, esto debería hacerse solo en proyectos en estado inicial, antes de cualquier modificación del mismo.
>
>Es posible crear un proyecto nuevo que incluya el *frontend* de autenticación usando el instalador de *Laravel* mediante `laravel new nombreProyecto --auth`.
>
>Sin embargo aquí no utilizaremos este *frontend*. Lo haremos desde cero, usando la *facade* ***Auth***.

La **configuración** del mecanismo de autenticación que viene con *Laravel* se halla en el archivo ***config/auth.php***.

Este archivo retorna un *array* que tiene varios elementos, cuyas claves son ***defaults***, ***guards***, ***providers***, ***passwords*** y ***password_timeout***.

- La clave ***providers*** describe los proveedores, es decir, las fuentes persistentes de las que se extraen los datos de usuario. Su valor es un *array* cuyos elementos son dichos proveedores: cada proveedor tendrá una clave con el nombre que daremos a ese proveedor, y su valor es un *array*:
  - El elemento con clave ***driver*** podrá tener el valor ***eloquent*** si el proveedor es un modelo *Eloquent*, o ***database*** si el proveedor es simplemente una tabla, sin modelo asociado.
  - El elemento ***model*** almacenará el nombre de la clase del modelo (si el proveedor es un modelo), o el nombre de la tabla.
- En ***guards*** definimos el guardián (método) de autenticación. Su valor es un *array* cuyos elementos son dichos *guards*: cada uno de ellos tendrá una clave con el nombre que daremos a ese *guard*, y su valor es un *array*:
  - El elemento ***driver*** indica el método de autenticación, que puede ser ***session*** o ***token***.
  - El elemento ***provider*** es el nombre de uno de los proveedores que hemos definido.
  - Se puede definir un elemento ***hash*** con valor ***false*** para indicar que no debe almacenar la contraseña *hashed*.
- El elemento ***defaults*** indica varias cosas, de las que solo nos interesa una: define cuál es el *guard* por defecto.

En cuanto a los otros elementos de la configuración, son más bien utilizados por los mecanismos de gestión de autenticación que genera *Laravel* automáticamente, con lo que no nos interesan.

Por defecto, *Laravel* proporciona un modelo ***User***, que se utiliza para acceder a la tabla de usuarios, con su correspondiente migración (tabla ***users***). Si queremos definir los campos de la tabla de usuarios nosotros mismos, hay que tener en cuenta que el campo ***password*** debe tener, como mínimo, 60 caracteres de anchura (si usamos `string()` sin especificar anchura será un ***VARCHAR*** de 255 caracteres, lo cual ya está bien). Por otro lado, debe contener un campo ***remember_token*** de 100 caracteres (para opción *remember me*).

### *Login*

Para autenticar a un usuario, se usa el método `attempt()` (intento de autenticación) de la *facade* ***Auth***, al que se le pasa un *array* con campos de la tabla de usuarios, para ver si tal usuario se encuentra en ella. Como mínimo, debería estar el campo que usemos para autenticarnos (nombre de usuario, normalmente) y la contraseña. Cuando esta está en el campo ***password***, *Laravel* le aplica un *hash* y la compara con la de la tabla (que está *hashed*). Si todo coincide, se inicia automáticamente una sesión autenticada. Este método, además, retorna ***true*** si la autenticación es correcta, o ***false*** en caso contrario.

```php
public function authenticate(Request $request)
{
    $credentials = $request->only('username', 'password');
    if(Auth::attempt($credentials))
        return redirect('dashboard');  // autenticación OK
    return redirect('inicio');  // autenticación fallida
}
```

Por otro lado, al método `attempt()` le podemos pasar en su lugar una serie de condiciones en un *array*:

```php
if(Auth::attempt(['username' => $usname, 'password' => $password, 'active' => 1]))
    // ...
```

Se iniciará la sesión autenticada y retornará ***true*** solo si se cumplen todas las condiciones (en este caso, también comprueba que el usuario esté marcado como activo).

### Uso de otros *guards*

Si queremos que la autenticación se realice mediante un *guard* que no sea el *default*, usaremos el método `guard()` antes de la llamada a `attempt()`:

```php
if(Auth::guard('un_guard')->attempt($credentials))
    // ...
```

### *Logout*

Para hacer *logout*, simplemente hay que invocar `Auth::logout()`.

### Recordar al usuario

Si queremos autenticar a un usuario activando la opción *remember me*, pasaremos a `attempt()` como segundo argumento (opcional) un booleano verdadero. De este modo, el usuario permanece logueado permanentemente (hasta que hace *logout* explícitamente). Este mecanismo utiliza el campo ***remember_token*** de la tabla o modelo de usuarios.

Es posible saber si un usuario logueado lo ha hecho a través de un *login* corriente o a través de un *remember me*, usando el método `Auth::viaRemember()`, que retorna ***true*** en el último caso.

### Otras formas de autenticación

Es posible autenticar a uno de los usuarios de la tabla. Simplemente se debe obtener el registro deseado de la tabla o modelo, y pasarlo al método `login()`:

```php
$user = User::where('username', 'manolito')->first();
Auth::login($user);
```

También se puede autenticar a un usuario por su clave primaria:

```php
Auth::loginUsingId(476378);
```

Esto métodos funcionan de forma similar a `attempt()` en cuanto a especificar el *guard* deseado, activar *remember me*, etc.

Por otro lado se puede autenticar a un usuario del mismo modo que si se tratase de `attempt()` pero solo para la *request* actual, de tal modo que no se generarían *cookies*, ni sesiones, ni nada (*stateless API*). Para ello se usaría el método `once()`.

### Obtener información de autenticación

Para saber si el usuario está autenticado, `Auth::check()` retornará ***true*** si lo está, y ***false*** si no.

Para obtener el usuario que está autenticado:

```php
$usuario = Auth::user();  // instancia  del usuario
$usuario_id = Auth::id();  // clave primaria del usuario autenticado
```

Al decir instancia del usuario nos referimos a un registro de la tabla (o modelo *Eloquent*) de usuarios.

Otra forma de obtener una instancia del usuario autenticado es mediante el método `user()` de la *request* actual.

Existen *middlewares* ya hechos en *Laravel* que pueden ser útiles, por ejemplo, a la hora de interceptar una *request* hacia una página por parte de un usuario no autorizado, por ejemplo. En todo caso podemos crear nuestro propio *middleware*, o controladores adecuados.

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

El primer argumento a `define()` es el nombre que damos a esa puerta. El segundo, es una *closure* que retornará ***true*** o ***false*** según se permita la acción o no. En nuestro caso, definimos una puerta relacionada con la acción de actualizar un *post* en un blog. La acción se permitirá si y solo si el usuario autenticado es el autor del *post*. Para ello, vemos que la *closure* recibe dos argumentos: en primer lugar, una instancia del usuario autenticado (que se le inyecta automáticamente). Después de este primer parámetro, vienen los otros argumentos que deseemos darle (en este caso, una instancia del *post*, que incluye información del autor).

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

Existe otro modo de usar las políticas, y es mediante directivas las *Blade* `@can`, `@cannot` y `@canany`.

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

El equivalente a `@can('update', $coche)` sería `@unless(Auth::user()->can('update', $coche))`.

También podemos comprobar si el usuario tiene por lo menos una de las capacidades listadas en un array mediante la directiva `@canany`:

```html
@canany(['update', 'view', 'delete'], $coche)
    <!-- para usuarios que pueden actualizar, ver o borrar el elemento $coche -->
@elsecanany(['create'], \App\Coche::class)
    <!-- para usuarios que no pueden hacer lo anterior pero sí crear un coche -->
@endcanany
```

Como hemos visto en los ejemplos, en acciones que no precisan de instancia del modelo (como `create`), se pasa el nombre de la clase.
