# Las bases

## Enrutado (*routing*)

Las rutas de la aplicación se hallan en la carpeta ***routes***, y se cargan automáticamente. Las rutas de la interfaz web se encuentran en ***routes/web.php***.

La definición de ruta básica acepta una *URI* y una *closure* (que pertenecen a la clase `Closure` de *PHP*) pasados a un método `get()` (o `post()`, `put()`, `delete()`, `patch()`, o `options()`) de la *facade* `Route`. Existen métodos para todos los verbos *HTTP*.

```php
Route::get('foo', function() {
    return 'Hello, world!'
});
```

Lo retornado por esta *closure* es precisamente la respuesta a la *request*. Si no es un objeto respuesta, *Laravel* se encarga de convertirlo en un objeto respuesta. En el caso del ejemplo, al hacer una petición ***GET*** de la *URI* ***foo***, simplemente obtendremos a la salida ***Hello, world!***.

El *service provider* ***RouteServiceProvider*** carga todas las rutas en el directorio ***routes***. El archivo ***web.php*** contiene las rutas de la interfaz web, y asocian al grupo de *middleware* web. En ***api.php*** están las rutas de la interfaz *API*, que al ser sin estado, no se asignan a ese grupo de *middleware*, sino al grupo *api*. En este caso, se le prefija ***/api*** a la *URI*.

Se puede definir una ruta que responda a varios verbos *HTTP* con el método `match()`:

```php
Route::match(['get', 'post'], '/foo', function() {
    return 'Hello, world!'
});
```

O a todos:

```php
Route::any('/foo', function() {
    return 'Hello, world!'
});
```

Es posible inyectar cualquier dependencia en los argumentos de la función *callback*, simplemente indicando el tipo de cada argumento.

### Protección *CSRF*

Para proteger de ataques *CSRF* (*cross-site request forgery*), los formularios que realicen peticiones a la aplicación deben contener un *token CSRF*. Esto solo se aplica en **rutas web** (no *API*) con métodos ***POST***, ***PUT***, ***PATCH*** y ***DELETE***.

Para insertar dicho *token* se puede usar la directiva de *Blade* `@csrf` dentro del formulario:

```html
<form method="POST" action="/foo">
    @csrf
    ...
</form>
```

Otra forma es enviarlo manualmente: se debe enviar al sevidor el campo ***_token*** del formulario con el valor del *helper* `csrf_token()`.

### Redirección

Se puede crear una redirección a otra *URI*:

```php
Route::redirect('/aqui', '/alli');
```

Normalmente la petición retornará un código 302 (redirección temporal). Para que retorne 301 (redirección permanente) tenemos dos opciones:

```php
Route::redirect('/aqui', '/alli', 301);
```

o

```php
Route::permanentRedirect('/aqui', '/alli');
```

### Vistas

Si lo único que hace la ruta es retornar una vista:

```php
Route::view('/welcome', 'welcome', ['name' => 'Taylor']);
```

El tercer argumento (parámetros de la vista) es opcional.

### Parámetros de la ruta

Para capturar un segmento de la *URI* (parámetros):

```php
Route::get('user/{id}', function($identidad) {
    return 'Usuario ' . $identidad;
});
```

Se pueden capturar tantos fragmentos (entre llaves) como se quiera. El nombre que se les de entre llaves es irrelevante, deben estar compuestos de caracteres alfabéticos y guión bajo (***\_***), pero no guión (***-***). Cada fragmento se inyectará en un argumento pasado a la función *callback*, en orden. Si además tenemos dependencias que debe inyectar el *service container*, estas deben ir (*type-hinted*) antes de los parámetros de la *URI*.

Si el nombre del fragmento entre llaves termina en ***?***, significa que el fragmento es opcional. En ese caso, se le debe dar un valor por defecto al parámetro correspondiente en la función *callback*.

Es posible restringir el formato de estos parámetros de la *URI* mediante expresiones regulares:

```php
Route::get('/user/{id}', function($id) {
    // ...
})->where('id', '[0-9]+');

Route::get('/user/{id}/{name}', function($id, $name) {
    // ...
})->where(['id' => '[0-9]+', 'name' => '[a-z]+']);
```

O mediante métodos como `whereAlpha()`, `whereNumeric()`, `whereAlphaNumeric()` o `whereUuid()`:

```php
Route::get('/user/{id}/{name}', function($id, $name) {
    // ...
})->whereNumber('id')->whereAlpha('name');
```

Si el parámetro no cumple con la restricción, se retorna un código 404.

### Rutas con nombre

Se pueden asignar nombres a las rutas. Esto se hace para simplificar las referencias a una ruta. Se hace encadenando el método `name()` a la definición de la ruta. Los nombres de las rutas deben ser únicos.

```php
Route::get('sections/user/profile', function() {
    /* ... */
})->name('profile');
```

Así, podríamos referirnos a esta ruta, simplemente por el nombre corto:

```php
$url = route('profile');  // obtenemos la URL
return redirect()->route('profile');  // redirigimos a esa ruta
```

Para obtener la *URL* podemos añadir parámetros, que serán añadidos a esta:

```php
$url = route('profile', ['id' => 1, 'photos' => 'yes']);
```

### Acceso a la ruta actual

Es posible acceder al objeto ruta actual, así como a la acción actual:

```php
$ruta = Route::current();  // ruta actual
$action = Route::currentRouteAction();  // acción de la ruta actual (string)
```

La ruta actual se puede obtener también a través del objeto *request* actual (se verá más adelante). Suponiendo que ***\$req*** sea una variable en la que se ha inyectado dicha *request*, podemos acceder mediante

```php
$ruta = $req->route();
```

### Grupos de rutas

Para especificar un grupo de rutas, podemos usar el método `group()` al que pasaremos una *closure* como argumento. Esta *closure* contendrá a su vez todas las definiciones de rutas deseadas (incluyendo otros grupos, si queremos).

```php
Route::group([], function() {
    Route::get(/* definición ruta */) -> name(/* nombre ruta */);
    Route::post(/* definición ruta */);
}) -> name(/* nombre del grupo */);
```

El primer argumento a `group()` es un *array* en el que se definen propiedades (atributos) comunes a todas las rutas del grupo. Por ejemplo, para añadir *middleware*, se asignará un *array* con nombres de *middleware* a la propiedad ***middleware***, etc.:

```php
Route::group(['middleware' => ['api', 'web'], 'prefix' => 'admin'], function() {
    // Definición de las rutas del grupo
});
```

Existe otra forma de crear grupos. En lugar de utilizar el método `Route::group()` con las propiedades en un *array*, podemos definir estas propiedades mediante métodos encadenados, siendo el método `group()` el último método de la cadena, en cuyo caso solo recibe solo un argumento (la *closure* con las definiciones de las rutas). Debe ser el último de la cadena porque retorna ***null***.

Para definir el *middleware* que afectará al grupo:

```php
Route::middleware(['miduno', 'middos'])->group(function () {
    // Definición de las rutas
});
```

Si solo afecta a una ruta:

```php
Route::get('/', function() {/*...*/})
    ->middleware('web');
```

El orden de encadenamiento suele ser indiferente, con lo que también podría hacerse:

```php
Route::middleware('web')
    ->get('/', function() {/*...*/});
```

Existen otros métodos para indicar propiedades para la ruta o grupo de rutas. Todos estos métodos pueden encadenarse a voluntad:

`controller()` recibe el nombre del controlador *fully qualified* asociado a la ruta. `domain()` recibe el nombre de un subdominio (acepta también parámetros):

```php
Route::domain('{account}.example.com')->group(function () {
    Route::get('user/{id}', function ($account, $id) {/*...*/});
});
```

Para añadir un prefijo a todas las rutas sin tener que teclearlo en todas, se usa `prefix()`, pasándole un *string* con dicho prefijo.

Si damos nombre con `name()` a un grupo de rutas, y en la definición de estas rutas también se les da nombre, el nombre del grupo se prefija al de la ruta. Si un grupo de rutas tiene nombre pero varias de sus rutas no lo tienen, deberemos ir con cuidado, ya que estas tendrán el misno nombre (el nombre del grupo).

Dar nombre a un grupo de rutas sin nombre es útil, por ejemplo, para asignar *middleware* a ese grupo a través de los archivos de configuración de *Laravel*.

### *Binding* de modelos

Es posible inyectar un modelo *Eloquent* directamente a partir de su ID en la *URL*:

```php
use App\Models\Coche;
// Binding implícito:
Route::get('/coches/{coche}', function (Coche $coche) {
    return $coche->marca;
});
```

También es posible inyectarlo en el controlador:

```php
Route::get('/coches/{coche}', [MiController::class, 'show']);
 // En la definición del método show():
public function show(Coche $coche) { /*...*/ }
```

Si en lugar del campo ID deseamos que se haga por otro campo, debemos indicarlo:

```php
Route::get('/coches/{coche:matricula}', function (Coche $coche) { /*...*/ });
```

Si queremos que el campo **por defecto** no sea ID, deberemos *override* el método `getRouteKeyName()` del modelo *Eloquent*:

```php
public function getRouteKeyName() {
    return 'matricula';
}
```

Por defecto se recibe una respuesta 404 si no se encuentra coincidencia.

### Ruta *fallback*

Con el método `fallback()` definimos la acción para las rutas que no hayan encontrado *match*. Normalmente será una acción de "página no encontrada":

```php
Route::fallback(function() { /*...*/ });
```

En este caso, no se especifica *URI* como primer argumento.

### *Method spoofing*

Dado que los formularios *HTML* solo permiten métodos ***GET*** y ***POST***, si deseamos enviar el formulario mediante otro método, usaremos la directiva *Blade* `@method()` a la que pasaremos un string con el método deseado (***PUT***, ***DELETE***, etc.).

### Aclaraciones sobre rutas

Las rutas mapean una *URI* a un recurso, como una vista, un controlador, un archivo, etc. Para ello, **todas** las *requests* tienen que llegar al *front controller* ***public/index.php***. Por lo tanto, es necesario que el servidor tenga las reescrituras habilitadas.

Al crear un proyecto, obtenemos las reescrituras adecuadas para el módulo ***rewrite*** de *Apache* en ***public/.htaccess***.

## *Middleware*

Se trata de componentes (filtros) por los que pasa toda petición *HTTP* antes de ser atendida, en función de la *URI*. Por ejemplo, el *middleware* que verifica si el usuario está autenticado, redirigirá el *request* hacia la página solicitada o hacia la página de *login*.

El *middleware* se ubica en ***app/Http/Middleware***. Para crear uno nuevo:

```
php artisan make:middleware NombreMiddleware
```

Esto creará la clase ***App\\Http\\Middleware\\NombreMiddleware*** en el archivo ***app/Http/Middleware/NombreMiddleware.php***.

*Laravel* irá ejecutando los *middlewares* sucesivamente. Para ello, si definen un método `handle()`, este será llamado. Este método define como primer parámetro, la *request*, y como segundo, una *closure* (de tipo ***Closure***) que deberemos usar para seguir con el *pipeline* de *middlewares*.

```php
public function handle($request, Closure $next)
{
    if ($request->age <= 200) {
        return redirect('home');
    }
    return $next($request);
}
```

Si decidimos que el *middleware* tiene éxito y siga la cadena, se debe retornar el resultado de la *closure* a la que se pasa el objeto *request*. En caso contrario, retornará algún tipo de respuesta (en el ejemplo, una redirección).

Habría que tener cuidado a la hora de redirigir. Si por ejemplo redirigimos a una ruta que usa el mismo *middleware* podríamos ocasionar una redirección cíclica infinita.

El ejemplo anterior realiza sus acciones **antes** de que la *request* sea atendida. Pero podemos hacer que realice acciones **posteriormente** a ser atendida. En ese caso, lo que deberemos retornar será la *response*, que obtendremos así:

```php
public function handle($request, Closure $next)
{
    // Acciones previas al tratamiento de la request
    $response = $next($request);    // invocación de la closure
    // Acciones posteriores al tratamiento de la request
    return $response;
}
```

### Registro de *middleware*

Para añadir el *middleware* de forma **global**, es decir, para que se ejecute siempre ante cualquier *request*, debemos registrar la clase del *middleware* añadiendo su nombre (*fully qualified*) en la propiedad ***$middleware*** de la clase en ***app/Http/Kernel.php***.

Por otro lado, la propiedad ***$routeMiddleware*** de la misma clase asocia una clave *string* a cada clase *middleware* (nombre completo). Podemos añadir un elemento a ese *array* con una clave relevante, y ya estaremos en condiciones de asociar ese *middleware* a una ruta o grupo de rutas mediante el método `middleware()`.

```php
Route::get('admin/profile', function () { /*...*/ })
    ->middleware('auth');
```

El método `middleware()` acepta un *array* de *strings*, un solo *string*, o un número arbitrario de ellos, de tal modo que podemos asociar un o más *middlewares* a una ruta (o grupo de rutas). Por otro lado, en lugar de la clave con la que se registró ese *middleware* podemos usar el nombre completo de la clase (así no habría necesidad de registrarla en ***$routeMiddleware***).

### Grupos de *middleware*

Se pueden crear grupos de *middlewares*. Esto se define en la propiedad ***$middlewareGroups***. Cada grupo tendrá una clave, de forma similar que se ha visto. El valor correspondiente a esa clave será un *array* con la lista de todas las clases *middleware* que queramos. Luego, al usar dicha clave en `middleware()`, se usarán todos los *middlewares* del grupo.

Por defecto, existen los grupos ***web*** y ***api***, que son registrados automáticamente por ***RouteServiceProvider***.

> También es posible registrar grupos de *middleware* mediante `middlewareGroup()`, que registra el grupo indicado. El primer argumento es el nombre del grupo, y el segundo es un *array* con nombres de *middleware*, de grupos de *middleware* o de clases *middleware*.
>
> Por otro lado, tenemos acceso también al método `getMiddlewareGroups()`, o `getMiddleware()`, que nos darán información sobre los grupos y nombres de *middleware* registrados.

### Exclusión de *middleware*

Si asignamos un *middleware* a un grupo, pero queremos que no afecte a una de las rutas del grupo (o a un grupo entero), se debe excluir ese *middleware* de esa ruta específica, mediante `excludeMiddleware()`, que admite los mismos parámetros que `middleware()`.

### Parámetros al *middleware*

Es posible pasar parámetros a nuestro *middleware*. Estos se recogen a partir del tercer parámetro del método `handle()` en adelante.

Para pasar los parámetros se haría mediante el método `middleware()` de la ruta, tras dos puntos (***:***), y separándolos por comas. En la definición de `handle()`:

```php
public function handle($request, Closure $next, $name, $age) { /* ... */ }
```

Y en la definición de la ruta:

```php
Route::get('cliente', function () {
    // Definición
})->middleware('auth:manolo,50');
```

Un *middleware* que esté registrado a nivel global no necesita definir parámetros (no tendría mucho sentido). Además, siempre nos queda acceder a la información de entrada que lleva la *request*.

## Controladores

Para no tener que definir el manejo de los *requests* como *closures* en los archivos de rutas, se pueden usar controladores, que son clases que permiten encapsular la lógica de tratamiento de peticiones. Los controladores están ubicados en ***app/Http/Controllers***.

Un controlador suele extender la clase ***App\\Http\\Controller***. Aunque no es obligatorio, si no lo hace se pierden algunas características de un controlador.

Si queremos asignar una ruta a un método específico de un controlador:

```php
Route::get('foo/{id}', [MiController::class, 'miMetodo']);
```

En este caso, se llamará al método `miMetodo()`, al que se la pasarán automáticamente como parámetro todos los parámetros de la *URI* (si los hay), en el orden en que aparecen.

> En versiones de *Laravel* anteriores a la 8, el segundo parámetro de `get()` era, en lugar de un *array*, un *string* del tipo ***'MiController@nombreMetodo'***.

Si el controlador contiene un método `__invoke()`, este será invocado por defecto si al definir la ruta no especificamos nombre de método (solo nombre del controlador):

```php
Route::get('foo', MiController::class);
```

Para crear un controlador:

```
php artisan make:controller nombreController
```

Si queremos que sea invocable (con método `__invoke()`), añadimos `--invokable`.

Por convenio, el nombre de una clase de controlador termina en "Controller".

### *Middleware* de un controlador

Normalmente asignábamos el *middleware* a una ruta o conjunto de rutas. Es posible asignar *middleware* a un controlador, independientemente de la ruta. Para ello se usará el método `middleware()` del controlador:

```php
class MiController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth');
        $this->middleware('log')->only('index');
        $this->middleware('subscribed')->except('store');
        $this->middleware(function ($request, $next) {
            // Función middleware
            return $next($request);  // por ejemplo
        });
    }
}
```

En este ejemplo, siempre que se invoque el controlador ***MiController***, se ejecutará el *middleware* con nombre ***auth*** (también se podría indicar el nombre de la clase *middleware*). En el caso de que el método (del controlador) que se vaya a ejecutar sea `index()`, se ejecutará también el *middleware* ***log***. Y en caso de que el método no sea `store()`, se ejecutará el *middleware* ***subscribed***. Por otro lado, la última sentencia indica que se ejecutará siempre el *middleware* definido por la *closure*, evitando así tener que crear una clase *middleware* completa.

Los métodos `only()` y `except()` aceptan un *string* o un *array* de *strings*.

### *Resource controllers*

Los controladores de recursos manejan todas las operaciones *CRUD* relacionadas con un recurso concreto (normalmente una tabla de una base de datos). Lo podemos crear a mano, o usar `artisan`:

```
php artisan make:controller CocheController --resource
```

A continuación, deberíamos definir las rutas para cada una de las operaciones *CRUD*, pero *Laravel* facilita mucho el trabajo permitiéndonos definir todas esas rutas en una sola línea:

```php
Route::resource('coches', CocheController::class);
```

Automáticamente, se acaban de definir estas rutas, las cuales, además, tienen un nombre asociado:

| Método HTTP | URI               | Método  | Nombre de la ruta |
| :---------- | :---------------- | :------ | :---------------- |
| GET         | /coches           | index   | coches.index      |
| GET         | /coches/create    | create  | coches.create     |
| POST        | /coches           | store   | coches.store      |
| GET         | /coches/{id}      | show    | coches.show       |
| GET         | /coches/{id}/edit | edit    | coches.edit       |
| PUT/PATCH   | /coches/{id}      | update  | coches.update     |
| DELETE      | /coches/{id}      | destroy | coches.destroy    |

El método `index()` sirve, básicamente, para la presentación de la tabla en pantalla; `create()` para presentar un formulario para introducir un elemento (registro) nuevo en la tabla; `store()` para insertar un elemento nuevo (con los datos enviados); `show()` debe retornar la información de un registro concreto para, por ejemplo, mostrar en pantalla; `edit()` para mostrar un formulario para editar un registro existente; `update()` para modificar un registro existente (con los datos enviados); y `destroy()` para eliminar un registro concreto.

Varios de estos métodos precisarán también de datos de entrada en la *request*. En cuanto a las *URIs* con parámetros (***id***), deberían contener información que permitiera identificar un registro concreto unívocamente (normalmente ***id*** será la clave primaria).

Si tuviéramos más de un *resource controller* podríamos incluir una línea para cada uno, o usar una sola sentencia, usando un *array*, de esta forma:

```php
Route::resource([
    'coches' => 'CocheController',
    'fotos' => 'FotoController'
]);
```

Deberemos crear todas las vistas que necesitemos, con los formularios adecuados. Cuando necesitemos un formulario que envíe datos mediante métodos distintos de ***GET*** y ***POST***, usaremos la directiva *Blade* `@method()`, como ya hemos visto.

A parte del método `middleware()`, los métodos `only()` y `except()` también se pueden usar en la definición de las rutas:

```php
Route::resource('coches', CocheController::class)->only(['index', 'show']);
```

### *API Resource controllers*

Podemos definir controladores de recursos que ofrecerán una *API*. En este caso, normalmente no desearemos definir algunas de las acciones mencionadas anteriormente, sobre todo las que presentan formularios para rellenar (concretamente las acciones `create()` y `edit()`).

Para crear las rutas de un *API resource controller*, que no incluirán los dos métodos mencionados, lo haremos mediante:

```php
Route::apiResource('coches', CocheController::class);
```

Estas *APIs* recibirán normalmente una petición (en consonancia con las rutas definidas arriba), y cuando sea necesario retornarán la información soliciatada. En el caso de una *API REST*, un objeto en formato *JSON*.

Normalmente definiremos las rutas, no en ***web.php***, sino en ***api.php***. La diferencia es que al definirlas en este último archivo, las *URI* van precedidas por ***api/***. Es decir, se definen automáticamente las rutas para ***api/coches***, ***api/coches/create***, etc.

Para retornar una *response* consistente en un objeto *JSON*, puede hacerse simplemente retornando un objeto del tipo *collection* desde el método concreto.

En todo caso, para crear el *API resource controller*, puede hacerse fácilmente mediante:

```
php artisan make:controller CocheController --api
```

Si lo que queremos es agrupar todos los controladores *API* en un directorio ***Api*** diferenciado, para mejor organización:

```
php artisan make:controller Api/CocheController --api
```

Al crear estos archivos, *Laravel* ya coloca las clases en los *namespaces* adecuados. En el último caso, la clase sería ***App\\Http\\Controllers\\Api\\CocheController***.

En todo caso, para comprobar todas las rutas definidas en el proyecto:

```
php artisan route:list
```

### Inyección de dependencias

Es posible inyectar cualquier dependencia en los parámetros de los métodos de un controlador. En este caso, las dependencias se inyectarán en los parámetros *type-hinted*, que deben ir antes de los posibles parámetros de la ruta.

## Peticiones (*requests*)

Existe un un objeto descriptivo de la *request* actual, al que se puede acceder a través de la inyección de dependencias, al hacer un *type-hint* de la clase ***Illuminate\\Http\\Request***. El *service container*, en este caso, nos enviará una instancia del objeto que describe la petición entrante. Cuando hablemos de la *request* nos referiremos normalmente a este objeto.

### Métodos de la *request* actual

El método `path()` retorna la *URI* actual. La *URI* no empieza por barra (***/***), a no ser que se refiera a la *URI* raíz, en cuyo caso consistirá en una simple barra.

El método `is()` comprueba si la *URI* coincide con cierto patrón que se le pasa como argumento (permite *wildcards*).

`url()` retorna la *URL* completa (incluyendo protocolo), pero sin la *query string*, mientras que `fullUrl()` la incluye. En el caso de `url()`, no terminará nunca con una barra, ya que las reescrituras por defecto eliminan la barra final.

Si además deseamos añadir parámetros extra a la *query string*:

```php
$ruta = $request->fullUrlWithQuery(['marca' => 'ACME', 'modelo' => '33']);
```

Para obtener el verbo (método) *HTTP*, `method()` retorna este. También existe `isMethod()`:

```php
if($req->isMethod('post'))
    /* ... */
```

Con el método `header()` podemos obtener el valor de una cabecera que esté incluida en la *request*. Si no existe tal cabecera, retornará ***null***, aunque en ese caso podemos especificar un segundo argumento con el valor por defecto, en caso de que no exista la cabecera:

```php
$valor1 = $request->header('Nombre-Cabecera1');
$valor2 = $request->header('Nombre-Cabecera2', 'Valor por defecto');
```

Para comprobar si existe una cabecera:

```php
if($request->hasHeader('Nombre-Cabecera'))
    /* ... */
```

Para obtener la *IP* del cliente, podemos usar el método `ip()`.

### Datos de entrada

Con el método `all()` se pueden obtener todos los datos de entrada (independientemente del método *HTTP*) en un *array*, tanto si se trata del envío de un formulario como de una petición *AJAX*.

En lugar de un *array* podemos obtener todos los datos de entrada en una *collection* mediante el método `collection`.

El método `input()` retorna un dato de entrada, cuyo nombre debemos especificar como argumento:

```php
$valor1 = $req->input('name');
$valor2 = $req->input('age', 20);
```

En el segundo caso, se le ha dado un valor por defecto. Si no se especifica, retornará ***null*** cuando no exista el campo.

Si el formulario contiene *arrays*, se usa la notación con punto (***.***) para acceder a estos:

```php
$valor = $req->input('products.0.name');
$valores = $req->input('products.*.name');
```

Si no indicamos ningún argumento, retornará un *array* con todos los datos.

Si solo deseamos los datos del *query string*, se usa el método `query()`, cuyo uso es igual al de `input()`.

Si queremos obtener datos *json* de una *json request*, usaremos el método `input()`. Para acceder al contenido se usará la notación con punto para indicar la jerarquía.

Si queremos obtener un valor de entrada como *booleano*, en lugar de `input()` usaremos el método `boolean()` al que pasaremos el nombre del campo. De forma similar, para fecha existe el método `date()`, que retornará una instancia de la clase ***Carbon***; los argumentos segundo y tercero (opcionales) indican formato y zona horaria.

Se puede usar una propiedad (dinámica) del objeto ***Request*** para acceder a los datos:

```php
$valor = $req->name;
```

Primero buscará un dato de entrada con ese nombre, y si no lo encuentra, lo buscará en los parámetros de la ruta, si los hay.

Si deseamos obtener solo un subconjunto de los datos de entrada, podemos usar los métodos `only()` o `except()`, a los que pasaremos la lista de los campos que deseamos obtener. La lista puede ser un número arbitrario de argumentos *string*, o un *array* con todos ellos.

Para saber si un dato concreto está presente:

```php
if($req->has('name')) ...
if($req->has(['name', 'age', 'height'])) ...
```

Como vemos en el ejemplo, si le pasamos un *array*, retornará ***true*** solo en el caso que **todos** los datos especificados estén presentes en la petición. En cambio:

```php
if($req->hasAny(['name', 'age', 'height'])) ...
```

Retornará ***true*** si por lo menos uno de ellos está presente.

El método `whenHas()` ejecuta la *closure* indicada como segundo argumento, siempre y cuando la entrada incluya el campo indicado en el primer argumento. El tercer argumento, opcional, es otra *closure* que se ejecutará si el campo no está incluido.

```php
$req->whenHas('name', function() {/*...*/}, function() {/*...*/});
```

El método `filled()` es como `has()` con un argumento no *array*, pero retornará ***false*** en el caso que el dato exista pero esté vacío.

El método `whenFilled()` es a `filled()` lo que `whenHas()` es a `has()`.

El método `missing()` es el inverso de `has()`.

Para añadir datos de entrada a la *request*, tenemos el método `merge()`, al que se pasará un *array* asociativo con todos los datos a añadir:

```php
$req->merge(['votes' => 0]);
$req->mergeIfMissing(['votes' => 0]);
```

En el segundo caso, se añadirá solamente si el campo no existe previamente.

#### Entrada antigua

*Laravel* permite mantener los datos de entrada de la *request* actual para la próxima *request*. Especialmente útil para repopular formularios en los que la validación ha fallado en el lado del servidor y deben volver a presentarse. Sin embargo, no suele ser necesario, ya que existen algunas características de **validación** proporcionadas por *Laravel* que ya ejecutan este mecanismo automáticamente.

El método `flash()` de la *request* flashea (inserta) **todos** los datos de entrada en la sesión, de tal modo que en el tratamiento de (únicamente) la subsiguiente petición estarán disponibles.

Si solo deseamos flashear algunos campos, se pueden usar los métodos `flashOnly()` o `flashExcept()`, a los que se pasará un *string* o un *array* de *strings* con el nombre o nombres de los campos.

Si se desea flashear los datos de entrada para redirigir a una página en concreto, se puede hacer utilizando los métodos `with()` o `withInput()`, como se verá en el apartado de respuestas de redirección.

Una vez estemos gestionando la *request* en la que existen datos flasheados de la petición anterior en la sesión, se podrá acceder a estos mediante el método `old()` de la *request*.

```php
$antiguoNombre = $req->old('name');
```

Sin embargo, en lugares (como una plantilla *Blade*) en los que no tenemos acceso al objeto *request*, es útil usar el *helper* `old()`:

```html
<input type="text" name="username" value="{{ old('user') }}">
```

Si no existe tal dato antiguo, `old()` (tanto el método como el *helper*) retornará ***null***, a no ser que le indiquemos como segundo argumento un valor por defecto.

Para obtener el valor de una *cookie*:

```php
$valor = $req->cookie('name');
```

### Archivos

Para acceder a un archivo subido en la actual *request*, se usa el método `file()`, al que se le pasa el nombre del campo correspondiente. También se puede acceder mediante propiedad dinámica:

```php
$archivo = $req->file('photo');
// Equivale a:
$archivo = $req->photo;
```

Para comprobar se un campo archivo viene con contenido, o para ver si el archivo subido es válido:

```php
if($req->hasFile('photo'))
    $archivo = $req->file('photo');
if $archivo->isValid() ...
```

El método `file()` retorna una instancia de ***Illuminate\\Http\\UploadedFile***. A parte de `isValid()`, esta clase dispone muchos otros métodos útiles. Podríamos mencionar `path()` (la ruta completa del archivo) o `extension()` (extensión del archivo basada, no en la extensión proporcionada, sino en el contenido).

Para almacenar el archivo, se usará el método `store()` del ***UploadedFile***. El primer argumento es la ruta en disco por defecto configurado en *Laravel*. Si en lugar de ese disco deseamos guardar el archivo en otro disco, especificaremos el nombre del mismo como segundo argumento. El método guardará el archivo con un nombre único generado automáticamente. El método retorna la ruta del archivo (incluyendo dicho nombre), relativa al raíz del disco concreto.

Si no deseamos que se guarde el archivo con un nombre autogenerado, usaremos el método `storeAs()`, al que se le debe pasar la ruta donde guardarlo, y el nombre deseado. Un tercer argumento opcional indicará el disco de destino.

Hay que tener en cuenta que para que se puedan enviar archivos desde un formulario *HTML*, la etiqueta `<form>` debe incluir el atributo ***enctype*** con valor ***multipart/form-data***, y que el método *HTTP* debe ser *POST*.

```html
<form action="subida.php" method="post" enctype="multipart/form-data">
```

## Respuestas (*responses*)

Ante cualquier *request*, toda ruta, a través de una *closure* o un controlador, debe retornar una *response*. No es necesario crear un objeto de tipo respuesta: lo que retornemos será convertido en respuesta automáticamente. Podemos retornar simplemente un *string*, o código *HTML* (directamente, o pasando una plantilla *Blade* al *helper* `view()`). También podemos retornar un *array* o *collection* (incluso un modelo *Eloquent*), que el *framework* convertirá en una respuesta *JSON*.

*Laravel* se encarga de construir la respuesta (con sus cabeceras, etc.) a partir de lo que nosotros retornamos.

El objeto respuesta es de tipo ***Illuminate\\Http\\Response***. Si queremos tener más control sobre esta respuesta, podemos construir el objeto y personalizarlo (código de respuesta, cabeceras *HTTP*, etc.):

```php
return response($contenido)
            ->header('Content-type', $type)
            ->header('Otro-header', 'valor otro header');
```

Esto equivale a:

```php
return response($contenido)
            ->withHeaders(['Content-type' => $type,
                           'Otro-header' => 'valor otro header']);
```

El *helper* `response()` acepta un primer argumento con el contenido de tal respuesta (*string*, vista, *array*, etc.). Como segundo argumento, opcional, el código de la respuesta (200 es *OK*). Opcionalmente puede tener un tercer argumento con un *array* de encabezados, aunque dichos encabezados se pueden añadir, como hemos visto, con el método `withHeaders()`.

También se pueden añadir *cookies* a la respuesta. Se hace de la misma forma que el método `header()`, pero en este caso con el método `cookie()`, cuyos tres primeros parámetros son los más usados: nombre, valor y tiempo de vida (en minutos).

> Los parámetros de este método son equivalentes a los del método `setcookie()` de *PHP*.

Alternativamente, en caso de no disponer de una instancia del objeto respuesta, se puede usar la *facade* ***Cookie*** para añadir *cookies* a la cola, que serán enviadas con la respuesta (mismos argumentos que el método `cookie()`:

```php
Cookie::queue(Cookie::make($nombre, $valor, 120));
// Equivale a:
Cookie::queue($nombre, $valor, 120);
```

En caso de querer eliminar una *cookie* de la respuesta, se puede usar el método `withoutCookie()`:

```php
return response($contenido)
           ->withoutCookie('nombre');
```

Alternativamente, si no tenemos acceso al objeto respuesta, podemos usar nuevamente la *facade* ***Cookie***:

```php
Cookie::expire('nombre');
```

Por defecto, las *cookies* están encriptadas y firmadas (no pueden ser modificadas por el cliente).

### Redirecciones

Una redirección es una respuesta de tipo ***Illuminate\Http\RedirectResponse***. Se trata de redirección externa (genera nueva *request* en el cliente). Para generar una redirección:

```php
Route::get('dashboard', function() {
    return redirect('home/dashboard');
});
```

Si hay que redirigir el usuario a su localización anterior, por ejemplo, si ha enviado un formulario inválido y hay que retroceder conservando los datos que ha tecleado:

```php
return back();  // así se perderían los datos tecleados
return back()->withInput();    // así se conservan
```

Dado que este mecanismo utiliza la sesión, solo funcionará si este código está en una ruta que utiliza el *middleware* ***web*** (las peticiones *API* no usan sesión alguna).

Lo que hace el método `withInput()` (con `back()`, o `redirect()`, etc.) es *flashear* **los valores de la entrada** (como hace el método `flash()` de la *request*), de tal modo que en la siguiente *request* estarán disponibles, como se ha visto, a través de `old()`.

Para redirigir a una ruta con nombre:

```php
return redirect()->route('login');
```

Si la ruta corresponde a una *URI* con parámetros, se puede añadir como segundo argumento, opcional, un *array* con estos:

```php
return redirect()->route('perfil', ['id' => 1]);
```

Para redirigir a la acción de un controlador:

```php
return redirect()->action([NombreController::class, 'metodo']);
```

Se puede pasar un segundo argumento con un *array* de parámetros de la *URI*.

Para redirigir a un sitio externo:

```php
return redirect()->away('https://www.somedomain.com');
```

Para añadir datos concretos a la sesión al redirigir:

```php
return redirect('dashboard')->with('estado', '¡Todo correcto!');
```

En el tratamiento de la redirección (probablemente una plantilla *Blade*), se tendrá acceso a esos datos *flashed*, mediante `session('estado')`.

### Otros tipos de respuesta

Si además de retornar una respuesta personalizada como las que hemos visto, el contenido debe ser una vista, hay que añadir esta mediante el método `view()`:

```php
return response()
            ->header('Content-type', $type)
            ->view('vistaHello', $data, 200);
```

En caso de que no queramos personalizar cabeceras, y simplemente queramos retornar una vista, podemos simplemente usar la función `view()`:

```php
return view('vistaHello', $data, 200);
```

Para una respuesta *JSON*, se puede usar el método `json()`, el cual establecerá automáticamente la cabecera ***Content-Type*** a ***application/json***, y convertirá el *array* proporcionado a formato *JSON* usando automáticamente la función de *PHP* `json_encode()`:

```php
return response()->json(['name' => 'Abigail', 'state' => 'CA']);
```

Si queremos que la respuesta haga que el cliente descargue un archivo:

```php
return response()->download($path, $name, $headers)->deleteFileAfterSend();
```

El único argumento obligatorio es el primero (ruta del archivo en el servidor). El segundo argumento es el nombre de archivo que verá el cliente, mientras que el tercer argumento es un *array* con cabeceras.

Si incluimos `deleteFileAfterSend()`, el archivo se eliminará del servidor una vez se haya producido una descarga correcta.

Si lo que queremos es que el archivo se muestre en el navegador, sin provocar una descarga a disco:

```php
return response()->file($path, $headers);
```

Los argumentos son la ruta del archivo en disco, y un *array* de cabeceras opcional.

## Vistas

Contienen el *HTML* servido, y separa la lógica de presentación de todo lo demás.

Se almacenan en ***resources/Views***, y tienen extensión ***.blade.php***.

Cuando definimos una ruta con el *helper* `view()`:

```php
Route::get('/', function() {
    return view('saludo', ['nombre' => 'James', 'edad' => 55]);
});
```

En este ejemplo, estamos retornando el contenido de la plantilla *Blade* ***resources/Views/saludo.blade.php***. En este caso estamos un segundo parámetro opcional con dos argumentos para la vista.

Si organizamos las vistas jerárquicamente, podemos usar *dot notation* en la especificación del nombre:

```php
Route::get('/', function() {
    return view('admin.saludo');
});
```

En este caso, retornamos la vista ***resources/Views/admin/saludo.blade.php*** (sin argumentos).

Debido a esta notación, los subdirectorios de las vistas no deben contener el carácter punto (***.***).

El *helper* `view()` equivale al método `make()` de la *facade* ***View***. Esta *facade* contiene otros métodos útiles. Por ejemplo, con el método `first()` se retorna la primera vista existente de una lista de vistas posibles:

```php
return View::first(['email.customer', 'customer', 'general.customer'], $args);
```

El *array* de argumentos es opcional.

También podemos comprobar la existencia de una vista:

```php
if(View::exists('email.customer.form'))
    /* ... */
```

### Paso de argumentos

Los argumentos se pueden pasar a una vista mediante un *array*, como se ha visto, o utilizando el método `with()`:

```php
return view('greeting')
           ->with('name', 'James')
           ->with('age', 55);
```

Es posible compartir datos entre todas las vistas de la aplicación, a través del método `share()` de la *facade* ***View***. Las llamadas para compartir datos se usan, típicamente, en el método `boot()` de un *service provider*:

```php
View::share('key', 'value');
```

### *View composers*

Se trata de código (*callback* o método) invocado cuando se renderiza una vista. Para **registrar** un *view composer* (normalmente mediante el método `boot()` de un *service provider*), se utiliza el método `composer()` de la *facade* ***View***:

```php
View::composer('greeting', GreetingComposer::class);
View::composer('farewell', function($vista) {
    /* ... */
});
```

El primer argumento es el nombre de la vista, y el segundo es el *view composer* (clase o *closure*).

*Laravel* no tiene una ubicación específica para las clases de *view composers*. Se pueden organizar en cualquier parte de la jerarquía (por ejemplo en ***app/View/Composers***).

En el ejemplo anterior, **justo antes** de renderizar la vista ***greeting***, se llamará al método `compose()` de la clase ***GreetingComposer***, que recibe como parámetro una instancia de dicha vista (de la clase ***Illuminate\\View\\View***). En el código del método podemos manipular el objeto vista, por ejemplo añadiéndole argumentos:

```php
public function compose(View $view)
{
    $view->with('count', 50);
}
```

Si definimos una *closure* como *view composer*, esta recibe como primer parámetro una instancia de la vista.

Si se quiere registrar un *view composer* para más de una vista, en lugar del nombre de la vista, se puede usar un *array* con los nombres de las vistas deseadas. A parte, se permite el uso de *wildcards*, con lo que, por ejemplo, asociar un *view composer* a ***\**** lo asociará a todas las vistas de la aplicación.

### Optimización

Las vistas se compilan y guardan para usos futuros. La versión compilada tiene preferencia cuando es posible presentarla (si la plantilla ha cambiado se vuelve a compilar). Pero estas compilaciones ocurren *on demand*, lo cual afecta negativamente al desempeño. Se puede ejecutar en línea de comandos:

```
php artisan view:cache
```

Esto compilará todas las vistas. Si se quiere borrar todo el caché de vistas:

```
php artisan view:clear
```

## Plantillas *Blade*

Se usan para definir las vistas de *Laravel*. Permiten extender un archivo *HTML* (incluyendo también, si se desea, fragmentos *PHP*) utilizando sus propias **directivas**.

### Datos

Como hemos visto, los argumentos a una vista se pasan de esta forma:

```php
return view('welcome', ['nombre' => 'Pepe']);
```

Para acceder a datos que se hayan pasado a la vista como argumento, se usa la plantilla`{{ $nombre }}`, donde el nombre de la variable debe coincidir con el nombre del argumento pasado. Esta plantilla equivale a un `echo` de *PHP*, y puede contener cualquier expresión *PHP*.

Las sentencias en una plantilla `echo` se pasan automáticamente por la función `htmlspecialchars()` de *PHP* para evitar ataques *XSS*. Así, por ejemplo, los caracteres que definen etiquetas *HTML* ***\<*** y ***\>*** quedan convertidos en ***\&lt;*** y ***\&gt;*** respectivamente, sientdo imposible inyectar código *HTML*.

Si no deseamos que se codifique una expresión, se debe usar en su lugar la plantilla `{!! $nombre !!}`. No se recomienda si la expresión son datos proporcionados por el usuario.

Existen *frameworks Javascript* que usan la plantilla `{{ ... }}`. Si queremos que *Blade* no lo tome como plantilla propia, sino que lo deje literalmente, se indicará como `@{{ ... }}`, de tal modo que lo único que hará *Blade* será eliminar la arroba (***@***), sin aplicar la plantilla *echo*. Si existe un fragmento con muchas de estas plantillas *Javascript*, se puede incluir dentro de una sección `@verbatim` ... `@endverbatim` para no tener que indicar la arroba en cada plantilla.

### Directivas

#### Sentencias condicionales

*Blade* dispone de la directiva `@if()` ... `@endif`, a la que se le debe pasar una expresión *PHP*. Dentro de esta se acepta la directiva `@elseif()`.

```
@if(count($records) < 10)
    Tengo poquitos.
@elseif(count($records) == 10)
    Tengo diez.
@else
    Tengo muchos.
@endif
```

Además, existe la directiva `@unless()` ... `@endunless`, que es lo contrario de `@if()`.

También se dispone de las directivas `@isset()` ... `@endisset` y `@empty()` ... `@endempty` que equivalen a las respectivas funciones *PHP*.

El código de la directiva `@auth` ... `@endauth` se utilizará si el usuario está autenticado, todo lo contrario que `@guest` ... `@endguest`. Opcionalmente, las directivas `@auth` y `@guest` aceptan un argumento con el *guard* usado para la autenticación (ver apartado de seguridad).

Para comprobar si una sección tiene contenido, `@hasSection()` ... `@endif`, pasándole como argumento el nombre de la sección. La inversa es `@sectionMissing()` ... `@endif`.

Cualquiera de estas directivas condicionales permite una sección final `@else`.

#### Sentencias switch

Para un mecanismo *switch*:

```
@switch($variable)
    @case(1)
        Primer caso...
        @break
    @case(10)
        Segundo caso...
        @break
    ...
    @default
        Caso por defecto...
@endswitch
```

#### Bucles

Existen varios tipo de bucles: `@for`, `@foreach`, `@forelse` y `@while`:

```
@for($i = 0; $i < 10; $i++)
    El valor es {{ $i }}
@endfor

@foreach($users as $user)
    <p>Este es el usuario {{ $user->id }}</p>
@endforeach

@forelse($users as $user)
    <li>{{ $user->name }}</li>
@empty
    <p>No users</p>
@endforelse

@while (true)
    <p>Looping forever</p>
@endwhile
```

`@forelse` es como `@foreach`, pero si el objeto sobre el que se va a iterar está vacío, se ejecutará únicamente la sección `@empty`.

Dentro de los bucles disponemos de las directivas `@break` y `@continue`, que son equivalentes al funcionamiento de sus contrapartes en *PHP*. Pero además admiten una condición, de tal modo que la directiva se ejecuta solo en el caso que la condición sea verdadera: `@continue($i < 1)`, o `@break($i > 100)`.

Dentro de un bucle tenemos acceso a la variable `$loop`, que nos da información sobre el estado del bucle. `$loop->first` es verdadero si estamos en la primera iteración, mientras que `$loop->last` es verdadero si estamos en la última.

Si el bucle está anidado, `$loop->parent` es la variable `$loop` del bucle padre.

`$loop->index` es el índice de la iteración actual (empieza en 0). `$loop->iteration` es la iteración actual (empieza en 1). `$loop->remaining` es la cantidad de iteraciones que quedan para terminar. `$loop->count` es el número de elementos del objeto que estamos iterando. `$loop->depth` indica el nivel de anidamiento del bucle. Las propiedades `$loop->even` y `$loop->odd` nos indican si el número de iteración es par o impar respectivamente.

### Subvistas

Mediante la directiva `@include()` se puede incluir una vista dentro de la actual. Todas las variables que estén disponibles en la plantilla actual, lo estarán también en la plantilla incluida. Se le pasa como argumento el nombre de la plantilla a incluir (con *dot notation*, como siempre). Como segundo parámetro se le puede pasar opcionalmente un *array* con parámetros extra para la plantilla a incluir.

Si la plantilla no existe, se levantará un error. Para evitarlo, se puede incluir mediante `@includeIf()`, la cual no levantará error si la plantilla especificada no existe.

La directiva `@includeWhen()` es como `@include()`, pero se inserta un primer parámetro con una condición, la cual, de evaluarse verdadera, realiza la acción de inclusión. Si la expresión no es verdadera, no se incluye la plantilla. Esta directiva es la contraria de `@includeUnless()`.

`@includeFirst()` es como `@include()`, pero en lugar de pasársele un nombre de plantilla, se le pasa un *array* con nombres de plantilla. Se incluirá la primera de ellas que exista.

### *PHP*

Es posible insertar bloques de código *PHP* usando la directiva `@php` ... `@endphp`.

### Comentarios

Se incluyen entre `{{--` y `--}}`, y no se insertan en el *HTML* resultante.

### Componentes

Un *Blade component* es un fragmento de vista altamente configurable. Se puede ver como un fragmento `<div>`, por ejemplo. Existen dos tipos de componentes: los que utilizan clases y los anónimos. Para crear un componente con una clase, se debe indicar el nombre de la clase:

```
php artisan make:component MensajeAviso
```

Este comando creará la plantilla *Blade* ***resources/views/components/mensaje-aviso.blade.php***. El nombre de la vista será ***components.mensaje-aviso***, aunque normalmente no se utilizará este nombre para el uso del componente. Para utilizarlo dentro de una plantilla, hay que incluir en esta:

```html
<x-mensaje-aviso/>
```

Obsérvese cómo se traduce la notación *PascalCase* de la clase a la *kebab-case* de la plantilla, así como el prefijo ***x-*** añadido al nombre de esta.

El comando de `artisan` creará también la clase ***App\\View\\Components\\MensajeAviso***, la cual extiende la clase ***Illuminate\\View\\Component***, en el archivo ***app/View/Components/MensajeAviso.php***.

Uno de los usos de esta clase, es la renderización del componente. Para ello, el método `render()` retorna dicha renderización. Por defecto, dicho método simplemente retorna la plantilla:

```php
public function render() {
    return view('components.mensaje-aviso');
}
```

Para crear un componente dentro de una jerarquía de directorios, se puede hacer:

```
php artisan make:component Mensaje/MensajeAviso
```

Este comando creará la plantilla ***resources/views/components/mensaje/mensaje-aviso.blade.php***, junto a la clase ***App\\View\\Components\\Mensaje\\MensajeAviso*** en el archivo ***app/View/Components/Mensaje/MensajeAviso.php***.

El nombre de la plantilla será ***components.mensaje.mensaje-aviso***, aunque para incluir el componente se hará así:

```html
<x-mensaje.mensaje-aviso/>
```

Si no deseamos la funcionalidad que aporta la clase de un componente, podemos crear un **componente anónimo**, es decir, únicamente se creará la plantilla, que funcionará de la misma forma, aunque sin las funcionalidades que podría aportarle la clase. Para ello, deberemos indicar el nombre de la plantilla (con la posible jerarquía en *dot notation*) y no el de la clase. Además indicaremos el *flag* `--view`:

```
php artisan make:component mensaje.mensaje-aviso --view
```

En este caso, también se incluirá el componente con `<x-mensaje.mensaje-aviso/>`. Su renderización no se realizará mediante un método `render()`, sino que se incluirá tal cual.

Al crear los componentes de esta manera, no es necesario registrar nada, ya que estas ubicaciones por defecto son descubiertas automáticamente por *Laravel*. Sin embargo, si utilizamos un paquete que tiene componentes, el *service provider* de este deberá incluir el registro de dichos componentes. Normalmente se hará en su método `boot()`:

```php
use Illuminate\Support\Facades\Blade;

public function boot()
{
    Blade::component('paquete-alerta', Alerta::class);
}
```

Esto registrará el nombre ***paquete-alerta*** con la clase asociada al componente, la cual contendrá el método `render()` pertinente. Una vez cargado el componente, se incluirá en una plantilla mediante `<x-paquete-alerta/>`.

#### Paso de datos

Es posible pasar argumentos a un componente, usando formato *HTML* al incluirlo:

```html
<x-mensaje-aviso tipo-ms="error" mensaje="No ha funcionado."/>
```

Estos parámetros son pasados directamente al constructor de la clase del componente, y deberían almacenarse dentro de la instancia para su posterior presentación.

El orden de aparición de estos parámetros en el constructor es indiferente, pero hay que tener en cuenta que los nombres deben coincidir. Además es importante recalcar que mientras los nombres de parámetros en la plantilla *HTML* deben estar en *kebab-case*, los del constructor deber tener *camelCase*.

```php
class MensajeAviso extends Component
{
    public $tipo;
    public $mensaje;

    public function __construct($tipoMs, $mensaje) {
        $this->tipo = $tipoMs;
        $this->mensaje = $mensaje;
    }

    public function render() {
        return view('components.mensaje-aviso');
    }
}
```

Al definir la plantilla del componente, esta tiene acceso a las propiedades (y métodos) de la clase, con lo que se puede escribir algo como esto:

```html
<div class="{{ $tipo }}">
    {{ $mensaje }}
</div>
```

Dichas propiedades deben tener acceso `public`, al igual que los métodos de la clase a los que se quiera acceder:

```html
<div class="{{ miMetodo($tipo) }}">
    {{ $mensaje . otroMetodo() }}
</div>
```

Podemos indicar que el valor de un parámetro no es un *string* en sí sino una expresión *PHP* prefijando dos puntos (***:***) al nombre del parámetro:

```html
<x-mensaje-aviso tipo-ms="error" :mensaje="'Error:' . $ms"/>
```

En numerosas ocasiones, la modificación de componentes y/o plantillas puede dar lugar a que *Laravel* produzca un error de ejecución debido a la cache de vistas compiladas. En ese caso resulta útil limpiar dicha cache mediante `php artisan view:clear`.

#### Acceso a atributos y *slots*

A parte de una vista, el método `render()` puede retornar un *string* con contenido (*componente inline*) o una *closure*. En este último caso, esta recibe un argumento único, consistente en un *array* con tres elementos: ***componentName*** (nombre del componente), ***attributes*** (atributos del componente) y ***slot*** (veremos más adelante qué es el *slot* de un componente).

```php
public function render()
{
    return function($datos) {
        // $datos['componentName'];
        // $datos['attributes'];
        // $datos['slot'];
        return '<p>Poca cosa.<p>';
    };
}
```

La *closure* en sí debe retornar el contenido a mostrar. El valor que contiene la clave ***attributes*** es un *array* con los atributos que se le han indicado mediante:

```html
<x-mensaje-aviso atri-uno="valor uno" atri-dos="valor dos"/>
```

Sin embargo, los **atributos que son recogidos por el constructor** de la clase no aparecerán en este *array*.

En el caso de este último ejemplo, para renderizar los atributos directamente en la plantilla:

```
<div {{ $attributes }}>
    <!-- Contenido -->
</div>
```

Esto daría lugar a:

```html
<div atri-uno="valor uno" atri-dos="valor dos">
    <!-- Contenido -->
</div>
```

En ente caso también, los atributos recogidos por el constructor de la clase tampoco serían incluidos.

Puede surgir un problema cuando el atributo se define tanto en la plantilla como en la invocación de esta. Por ejemplo, si escribimos la plantilla:

```
<div {{ $attributes }} class="uno">
    <!-- Contenido -->
</div>
```

Y lo invocamos así:

```html
<x-mensaje-aviso class="dos"/>
```

Asumiendo que el atributo ***class*** no es recogido por el constructor de la clase del componente, esto dará lugar a:

```html
<div class="dos" class="uno">
    <!-- Contenido -->
</div>
```

Lo cual es *HTML* incorrecto. Para solucionarlo, podemos usar el método `merge()`:

```
<div {{ $attributes->merge(['class' => 'uno']) }}>
    <!-- Contenido -->
</div>
```

Ahora, los atributos quedarán mezclados correctamente:

```html
<div class="uno dos">
    <!-- Contenido -->
</div>
```

Hay que aclarar que este método está pensado para mezclar atributos de la propiedad ***class***. Para otro tipo de propiedades, simplemente especifica su valor por defecto: Si en la invocación del componente se proporciona su valor, este tendrá preferencia sobre el valor indicado en `merge()`.

Para mezcla de clases (***class***) existe el método `class()`, que acepta un *array* con las clases a incluir/mezclar. Si el elemento tiene una clave numérica, la clase (o conjunto de clases) se añade siempre. En cambio, si la clave es un *string*, es la clave la que define la clase (o clases), y el valor es una expresión booleana, de tal modo que la inclusión se producirá solamente en caso de que la expresión sea cierta.

```
<div {{ $attributes->class(['uno', 'dos tres', 'cuatro cinco' => $expresion, 'seis']) }}>
    <!-- Contenido -->
</div>
```

Los métodos `merge()` y `class()` pueden encadenarse a voluntad.

Existen algunas palabras reservadas que no pueden usarse como propiedades públicas o métodos del componente: ***data***, ***render***, ***resolveView***, ***shouldRender***, ***view***, ***withAttributes*** y ***withName***.

#### Slots

Los *slots* son el contenido definido en el momento de incluir el componente. En lugar de:

```html
<x-mensaje-aviso/>
```

Se invoca así:

```html
<x-mensaje-aviso>
    <!-- Todo este contenido es el slot -->
</x-mensaje-aviso>
```

Todo lo que queda entre las etiquetas de inicio y cierre es el *slot*. En la plantilla que define el componente, el contenido completo del *slot* (incluyendo comentarios *HTML*) queda disponible mediante la variable ***\$slot***.

La plantilla podría definirse así:

```
<div id="33">
    {{ $slot }}
</div>
```

A parte de definir el contenido del *slot* en sí, que se recibe en la variable ***\$slot***, podemos definir también otras variables. Se puede ver como un pase de argumentos a la plantilla:

```html
<x-mensaje-aviso>
    <x-slot:titulo>
        Este es el título
    </x-slot>

    <x-slot:subtitulo>
        Este es el subtítulo
    </x-slot>

    <!-- Contenido del slot -->
</x-mensaje-aviso>
```

En la plantilla, recibiremos ese contenido en las variables correspondientes:

```
<div id="33">
    <h1>{{ $titulo }}</h1>
    <h3>{{ $subtitulo }}</h3>
    {{ $slot }}
</div>
```

Adicionalmente a definir estas variables, es posible asignarles atributos:

```html
<x-mensaje-aviso>
    <x-slot:titulo class="uno">
        Este es el título
    </x-slot>
    <!-- Contenido del slot -->
</x-mensaje-aviso>
```

En la plantilla, tendremos acceso a la propiedad ***\$attributes*** de esa variable del *slot*, que puede usarse del mismo modo que la propiedad ***\$attributes*** del componente, incluyendo el uso de los métodos `merge()` y/o `class()`.

```
<div id="33">
    <h1 {{ $titulo->attributes }}>
        {{ $titulo }}
    </h1>
    {{ $slot }}
</div>
```

### Construcción de *layouts*

Para construir el *layout* de nuestra página, podemos hacerlo mediante componentes, o usando extensión de plantillas.

#### *Layout* con componentes

Para ello habrá que crear un componente *layout* que admita distintos parámetros. A la hora de presentarlo, usaremos una vista que incluirá ese *layout*, definiendo los valores del posible *slot*, variables, etc.

#### *Layout* con herencia de plantillas

`@extends` sirve para heredar de una plantilla base (padre). Por ejemplo `@extends('layout.app')` extenderá la plantilla ***resources/views/layout/app.blade.php***.

Esto es, hace que la plantilla hija herede el contenido de la plantilla base. Pero para personalizar el contenido heredado, debemos definir una serie de **secciones**, o lugares donde la plantilla base espera que se le pase contenido. Las secciones las definimos con la directiva `@section()` en la plantilla hija, mientras la plantilla base muestra ese contenido mediante la directiva `@yield()`.

Supongamos que en un punto de la plantilla base, se incluye `@yield('titulo')`. Cuando nosotros extendamos esa plantilla base, deberemos definir, en la plantilla hija, el contenido de la sección ***titulo***. Esto se podría hacer así:

```
@section('titulo', 'Página de bienvenida')
```

Esto simplemente sustituye la llamada a `@yield()` con el texto ***Página de bienvenida***.

Si queremos expandir varias líneas en lugar de un simple *string*, disponemos de esta sintaxis alternativa:

```
@section('display')
    <h1>Esta es la sección display</h1>
    <!-- Más contenido para la sección -->
@endsection
```

Es posible también que la plantilla base tenga alguna sección definida con `@section`. Esto sucede cuando extiende a su vez otra plantilla base. En el caso de una jerarquía de varios niveles en la que existe una sección definida con el mismo nombre en varios niveles consecutivos, la definida en el nivel más bajo tendrá preferencia, sobrescribiendo las demás. Para incluir dentro de la sección la definición en la plantilla de nivel superior, se incluirá la directiva `@parent`.

En el caso de que la plantilla base defina una sección, si esta plantilla no extiende otra plantilla, la sección quedaría definida únicamente, y no se mostraría. Si queremos que la sección que definimos quede *yielded* inmediatamente, en lugar de terminarla con `@endsection` la terminaremos con `@show`.

La directiva `@yield()` acepta un segundo argumento opcional, que usará si no encuentra la sección especificada.

## Generación de *URLs*

Dada una *URI*, la función `url()` construye una *URL* completa, añadiendo el esquema (*http* o *https*) y el nombre del *host*, correspondientes a la *request* actual. Si no se le pasa argumento, retornará un **objeto** que describe la *URL* actual. A partir de este, podemos acceder a la siguiente información:

```php
url()->current();    // URL actual, sin la query string
url()->full();    // URL actual completa
url()->previous();    // URL de la request anterior
```

Esto se puede hacer igual con la *facade* ***URL***:

```php
URL::current();
URL::full();
URL::previous();
```

### *URLs* de rutas con nombre

Supongamos la definición de la siguiente ruta:

```php
Route::get('/productos', /* definición de la ruta */ )
    ->name('prods');
Route::get('/clientes/{numero}', /* definición de la ruta */ )
    ->name('clients');
```

Suponiendo que la aplicación está en ***https://miservidor.com***, la expresión `route('prods')` retornará ***https://miservidor.com/productos***. En cambio, no podemos hacer `route('clientes')` porque debemos indicar el parámetro de la *URL*. Esto debe hacerse en el segundo argumento a `route()`, en el cual definiremos todos estos parámetros en un *array*. Si en el *array* definimos parámetros extra que no son parámetros de la *URL*, los añadirá en la *query string*:

```php
echo route('clients', ['numero' => 33, 'modo' => 'full']);
```

Esto producirá ***https://miservidor.com/clientes/33?modo=full***.

### *URLs* de acciones de controlador

Podemos obtener una *URL* para un método concreto de un controlador con la función `action()`:

```php
$url = action([NombreController::class, 'metodoX']);
```

Si el controlador es *invokable*, podemos hacer:

```php
$url = action(NombreController::class);
```

Cualquiera de las dos formas aceptará también un segundo argumento con un *array* de argumentos de la *URL*.

## Sesión

La configuración de sesión está en ***config/session.php***. Por defecto, *Laravel* utiliza el *driver* de sesión ***file*** (está establecido en la opción de configuración ***driver***). Los posibles *drivers* son:

- ***file*** - sesiones almacenadas en ***storage/framework/sessions***.
- ***cookie*** - se almacenan en *cookies* encriptadas.
- ***database*** - almacenadas en una base de datos relacional.
- ***memcached***, ***redis*** - almacenadas en cachés en memoria rápidos.
- ***array*** - en un *array PHP* (sin persistencia).

### Uso de la sesión

Se puede acceder a los datos de sesión mediante el *helper* `session()` o mediante la instancia de la ***Request***. En este último caso, el acceso es de este tipo:

```php
public function show(Request $req) {
    return $req->session()->get('clave');
}
```

A este método `get()` se le puede pasar un segundo argumento opcional, que será devuelto en caso de que la clave solicitada no exista en la sesión. Este argumento opcional puede ser una *closure*, que será ejecutada y se usará el valor que retorne.

También se puede usar la función global `session()`, cuyo uso es igual, pero en este caso podemos establecer valores de sesión pasándole como argumento un *array* con pares clave/valor en lugar de un *string*.

Para obtener todos los datos de sesión en un *array*:

```php
$data = $req->session()->all();
```

Para determinar si un determinado dato existe:

```php
if($req->session()->has('key'))
    // ...
```

Retorna ***true*** si el elemento existe y no es ***null***. Similar al método `exists()`, que será verdadero incluso aunque el valor sea  ***null***.

Por otro lado, el método `missing()` retorna verdadero si el elemento es ***null*** o no existe (es el inverso de `has()`).

Para almacenar un dato, se puede hacer de una de las dos formas:

```php
$req->session()->put('clave1', 'valor1');
session(['clave2' => 'valor2']);
```

Si el valor correspondiente a una clave es un *array*, puede insertarse un nuevo elemento en dicho *array* mediante `push()`:

```php
$req->session()->push('user.teams', 'developers');
```

Esto añade un valor al *array* cuya clave es ***user.teams***.

Un método equivalente a `get()` es `pull()`, pero en este caso, si la clave existe la borra de la sesión después de retornarla.

### *Flashing* de datos de sesión

Para que ciertos datos estén disponibles en la *request* actual y también en la siguiente (pero no más allá), se hace mediante el método `flash()`.

```php
$req->session()->flash('status', 'Mensaje enviado con éxito');
```

En este caso, el elemento ***status*** estará disponible también durante la siguiente *request*. Si en la siguiente solicitud decidimos prorrogar la disponibilidad de los datos *flashed* una *request* más:

```php
$req->session()->reflash();
```

Y si solo deseamos hacerlo con un subconjunto de los datos *flashed*:

```php
$req->session()->keep(['clave1', 'clave2', 'clave3']);
```

Si lo que queremos es que un dato concreto esté disponible únicamente durante la *request* actual, usaremos el método `now()` en lugar de `flash()`.

### Eliminación de datos

Para eliminar datos de la sesión:

```php
$req->session()->forget('key');    // elimina un dato de sesión
$req->session()->forget(['key1', 'key2', 'key3']);    // varios datos
$req->session()->flush();    // todos los datos
```

## Validación

Validación de los datos de entrada (formulario) que llegan al servidor. Lo más utilizado es el método `validate()` del objeto ***Request*** actual. Al invocar este método, se le deben pasar las reglas de validación en un *array*:

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
]);
```

Si la validación falla, el código posterior a `validate()` no se ejecuta, y se retorna automáticamente una redirección a la *URL* anterior (la presentación del formulario). En cambio, si la validación es correcta, podemos seguir con el código para tratar los valores recibidos del *submit*.

El valor de retorno del método `validate()` es un *array* asociativo con los datos validados.

Las reglas de validación pueden pasarse como un *array* también: `['required', 'unique:posts', 'max:255']`. Véase la documentación oficial para una explicación de los tipos de validación disponibles.

> Los campos opcionales deberían ser marcados como ***nullable*** en la validación, ya que *Laravel*, por defecto convierte las cadenas vacías en ***null***.

En caso de datos de entrada anidados, se puede usar la *dot notation* para hacer referencia a ellos. Es por esto que si el nombre del campo contiene un punto, debe indicarse *escaped*.

### Mostrar los errores de validación

Cuando la validación ha fallado, y se recibe la respuesta de la posterior redirección, los datos de entrada antiguos son *flashed* (accesibles mediante el *helper* `old()`, como se ha visto anteriormente). En realidad, la redirección retornada automáticamente por el sistema de validación equivale a `back()->withInput()`.

Si por ejemplo deseamos un valor por defecto cuando no existe valor antiguo (*old data*), se puede hacer así:

```html
<input type="text" name="ciudad" id="ciudadform" value="{{ old('DNI') ?? 'Sabadell' }}">
```

Así, en el caso de que el valor antiguo exista, retornará este; de lo contrario, será ***Sabadell***. Esto muestra el uso del operador `??` de *Blade* (similar al operador ternario).

Por otro lado, los errores producidos son enviados como argumento a todas las vistas de la aplicación (cuando tratamos la redirección por fallos de validación), a través de la variable `$errors` (de tipo ***Illuminate\Support\MessageBag***). En la plantilla *Blade* se puede comprobar su contenido:

```
@if($errors->any())
  <ul>
    @foreach ($errors->all() as $error)
      <li>{{ $error }}</li>
    @endforeach
  </ul>
@endif
```

Alternativamente puede usarse la directiva *Blade* `@error`, y la variable ***$message***:

```
@error('title')
  <p>{{ $message }}</p>
@enderror
```

En cuanto a los mensajes de error, estos se encuentran definidos en ***lang/en/validation.php***. Pueden cambiarse a voluntad, así como crear versiones de este archivo traducidas a otro lenguaje.

> Si la *request* inicial era asíncrona, el método `validate()` no retornará una redirección sino una respuesta *JSON* conteniendo todos los errores de validación, acompañada de un código *HTTP* 422.

### Validación con *form requests*

Un *form request* es una clase que encapsula la lógica de validación y autorización. Se puede crear así:

```
php artisan make:request MiRequest
```

Esto creará la clase ***App\\Http\\Requests\\MiRequest***, en el archivo ***app/Http/Requests/MiRequest.php***. Esta clase dispone de los métodos `authorize()` y `rules()`.

Para que se ejecute este método de validación **no hay que escribir absolutamente nada de código**. Lo único que hay que hacer es inyectar la clase de la *form request* creada en el método de nuestro controlador donde queramos que se produzca la validación. Si no pasa la validación, el método no se ejecutará. Si falla, se procederá como en el caso de `validate()` (redirección o respuesta *JSON*).

La *form request*, de hecho, extiende la clase ***App\\Http\\Request***, de tal modo que la instancia contiene toda la información de la *request* actual.

El método `authorize()` comprueba si el usuario autenticado tiene permisos para realizar la presente acción. Puede accederse a dicho usuario mediante `$this->user()`. Si el método retorna ***false***, se retornará una respuesta 403. En cambio, si retorna un valor verdadero, se seguirá adelante con las validaciones. Para ello se utiliza el valor de retorno del método `rules()`, que de hecho retorna el *array* de validación, que es el que usa el método `validate()`.

Si a nuestra *form request* añadimos una propiedad ***\$stopOnFirstFailure*** establecida en ***true***, tras encontrar el primer error de validación ya no se evalúan los siguientes campos (es equivalente a la validación ***bail***).

Por defecto se hace una *redirección* `back()`, pero mediante la propiedad ***\$redirect*** definimos la *URI* donde redirigir. También podemos definir la redirección dándole un nombre de una ruta con nombre a la propiedad ***\$redirectRoute***.

### Creación manual de validadores

Es posible crear un objeto validador, el cual disponga del método `validate()`. Aunque en el objeto *request* disponemos ya de un validador, crear uno a medida proporciona mayor control y mucha más funcionalidad. Para más información, véase la documentación oficial.

### Trabajar con los datos validados

Cuando la validación ha sido correcta, tenemos una serie de datos que han sido validados, pero puede haber otros campos que no hayan pasado por la validación (por no ser necesario, o porque se han insertado de forma ilícita). Existe una forma de acceder al subconjunto de datos que han pasado por la validación: por un lado, el método `validate()` de la *request*, y por otro, el método `validated()` de una *form request* (no de la *request*) o de un validador creado manualmente. Estos métodos retornan un *array* con los datos.

En lugar de este método puede usarse `safe()`, el cual retorna un objeto de tipo ***Illuminate\Support\ValidatedInput***. Este objeto dispone de tres métodos, los cuales retornan un *array* con un subconjunto de los datos validados: `all()` los retorna todos, `only()` retorna solo los que le indicamos en un argumento *array*, y `except()` hace lo contrario. Además, este objeto puede ser iterado y accedido igual que un *array*. También puede convertirse en una *collection* usando el método `collect()`.

### Reglas de validación personalizadas

Si las reglas proporcionadas no son suficientes y deseamos añadir una nueva regla de validación (*custom validation rule*), podemos crear un nuevo objeto regla así:

```
php artisan make:rule MiRule
```

Esto crea la clase ***App\\Rules\\MiRule*** en el archivo ***app/Rules/MiRule.php***. La regla tendrá los métodos `passes()` y `message()`. El primero recibe el nombre del atributo y su valor (2 parámetros), y debe retornar ***true*** o ***false*** según pase la validación o no. Por su parte, el método `message()` debe retornar el mensaje de error en caso de que falle la validación.

Para utilizar la regla, simplemente se debe pasar una instancia a la lista de reglas para un campo concreto:

```php
$req->validate([
    'DNI' => ['min:8', 'max:9', 'required', new MiRule],
    'Nombre' => 'required'
]);
```

En lugar de crear dicha regla, podemos simplemente utilizar una *closure*, la cual recibe tres parámetros: el nombre del atributo, su valor y otra *closure*, que debe ser llamada en caso de fallo (si no lo hace, se entiende que la validación es correcta).

```php
$req->validate([
    'DNI' => [
        'min:8',
        'max:9',
        'required',
        function($atributo, $valor, $fallo) {
            if($valor == '666')
                $fallo('El campo ' . $atributo . ' no puede ser 666...');
        }],
    'Nombre' => 'required'
]);
```

Cuando un campo no está presente o está vacío, las reglas de validación (incluyendo las *custom rules*) no se ejecutan en él. Si queremos crear una *custom rule* que se ejecute aunque el campo no esté presente o esté vacío, la crearemos con *artisan* añadiendo el *flag* `--implicit`.

## Tratamiento de errores

Las excepciones levantadas durante la ejecución se loguean y muestran a través de la clase ***App\Exceptions\Handler***. Esta clase permite tener control acerca de cómo y qué se reportará en caso de fallo.

En el archivo de configuración ***config/app.php*** existe una clave, ***debug***, con un valor booleano que indica la cantidad de información que se muestra al usuario en caso de error. Esta variable, por defecto, lee el contenido de ***APP_DEBUG*** en el archivo ***.env***.

### Excepciones *HTTP*

Es posible retornar un código de error *HTTP* mediante el *helper* `abort()`, pasándole como argumento el código específico.

Por otro lado, es posible definir páginas asociadas a un *status code* concreto. Estas páginas deben estar en el directorio ***resources/views/errors***, y tendrán como nombre el código deseado. Por ejemplo, para el estado de página no encontrada, podríamos definir el archivo ***resources/views/errors/404.blade.php***. También es posible definir una página para una serie de códigos. Esto se consigue con las plantillas ***4xx.blade.php*** (códigos 400) y ***5xx.blade.php*** (códigos 500). Estos archivos deben residir en el directorio indicado anteriormente. En todos estos archivos, está disponible la variable ***\$exception***, que contiene la excepción *HTTP*, y es del tipo ***Symfony\Component\HttpKernel\Exception\HttpException***. Se puede acceder al mensaje de error mediante su método `getMessage()`.

*Laravel* ofrece una serie de plantillas de error por defecto, que pueden publicarse y personalizarse adecuadamente. Para publicarlas:

```
php artisan vendor:publish --tag=laravel-errors
```

## *Logging*

*Laravel* usa la biblioteca *Monolog* para realizar el *logging*.

La configuración de *logging* se encuentra en ***config/logging.php***.

El *array* que retorna este archivo tiene una clave ***default*** con el nombre del canal por defecto.

La clave ***deprecations*** indica qué canal se usará para registrar mensajes relativos a *deprecation* de características y bibliotecas. Alternativamente se puede definir un canal con el nombre ***deprecations***. De una u otra forma, los mensajes mencionados se registrarán en este canal automáticamente.

Otra clave importante es ***channels***, que contiene un *array* cuyos elementos definen los diferentes canales de *logging*. Cada uno de estos elementos consta de una clave con el nombre del canal, y como valor un *array* con las distintas configuraciones del mismo.

Dentro de la configuración de cada canal, la clave ***driver*** define el tipo de *log* de dicho canal. El resto de claves dependen de dicho tipo y configuran el canal.

Algunos tipos de *drivers* disponibles por defecto son (clave ***driver***):

- ***stack***: en este caso, una clave ***channels*** tendrá como valor un *array* con los nombres de los canales que forman dicho *stack*.
- ***single***: se refiere a un simple archivo de *log*, definido en la clave ***path***. La clave ***level*** define el nivel mínimo de *logging*.
- ***daily***: es como ***single***, pero con *logging* rotativo (el número de días se indica en ***days***).

Para otros tipos de canal, véase la documentación oficial.

Los *logs* ***single*** y ***daily*** admiten también los siguientes campos:
- ***bubble***, por defecto ***true***, indica si el mensaje se propaga por el posible *stack* (en caso de formar parte de uno).
- ***permission*** indica los permisos por defecto del archivo (por defecto 0644).
- ***locking***, por defecto ***false***, indica si el archivo debe bloquearse antes de escribir en él.

Los niveles de *logging* son, en orden descendente, ***emergency***, ***alert***, ***critical***, ***error***, ***warning***, ***notice***, ***info***, y ***debug***. Cada canal define su nivel mínimo.

Para enviar mensajes al *log*, se usará la *facade* ***Illuminate\Support\Facades\Log***, utilizando los métodos oportunos, cuyo nombre coincide con el nivel de *logging*.

```php
Log::info($mensaje);
```

Para enviar también información al *array* de contexto, pasaremos tal *array* como segundo argumento del método (en este caso `info()`).

En el ejemplo, el mensaje será enviado al canal por defecto. Para enviarlo a cualquier otro canal:

```php
Log::channel('nombre_canal')->info($mensaje);
```

Si deseamos crear un *stack* sobre la marcha para registrar un mensaje:

```php
Log::stack(['single', 'monolog'])->info('Mensaje a registrar');
```

Si el nivel del mensaje es inferior al nivel definido en cada *log* específico, no se registrará.

También podemos crear un canal sobre la marcha para registrar un mensaje:

```php
Log::build([
    'driver' => 'single',
    'path' => storage_path('logs/custom.log'),
])->info('Mensaje a registrar');
```

Podemos incluso crear un *stack* sobre la marcha al que podemos pasarle un canal creado sobre la marcha:

```php
$canal = Log::build([
    'driver' => 'single',
    'path' => storage_path('logs/custom.log'),
]);

Log::stack(['monolog', $canal])->info('Mensaje a registrar');
```
