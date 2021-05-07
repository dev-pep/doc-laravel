# Las bases

## Enrutado (*routing*)

Las rutas de la aplicación se hallan en la carpeta ***routes***, y se cargan automáticamente. Las rutas de la interfaz web se encuentran en ***routes/web.php***.

La definición de ruta básica acepta una *URI* y una *closure* pasados a un método `get()` (o `post()`, `put()`, `delete()`, etc.) de la *facade* `Route`. Existen métodos para todos los verbos *HTTP*.

```php
Route::get('foo', function() {
    return 'Hello, world!'
});
```

En este caso, al hacer una petición ***GET*** de la *URI* ***foo***, simplemente obtendremos la salida ***Hello, world!***.

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

### Protección CSRF

Para proteger de ataques *CSRF* (*cross-site request forgery*), los formularios que realicen peticiones a la aplicación deben contener un *token CSRF*. Para ello se puede usar la directiva de *Blade* `@csrf` que genera este *token*:

```html
<form method="POST" action="/foo">
    @csrf
    ...
</form>
```

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

Para capturar un segmento de la *URI*:

```php
Route::get('user/{id}', function($identidad) {
    return 'Usuario ' . $identidad;
});
```

Se pueden capturar tantos fragmentos (entre llaves) como se quiera. El nombre que se les de entre llaves es irrelevante, deben estar compuestos de caracteres alfabéticos y signos, sin guión (***-***). Cada fragmento se corresponderá con un argumento pasado a la función *callback*, en orden.

Si el nombre del fragmento entre llaves termina en ***?***, significa que el fragmento es opcional. En ese caso, se le debe dar un valor por defecto al parámetro correspondiente en la función *callback*.

### Ruta *fallback*

Con el método `fallback()` definimos la acción para las rutas que no hayan encontrado *match*. Normalmente será una acción de "página no encontrada":

```php
Route::fallback(function() { /* ... */ });
```

### Acceso a la ruta actual

Es posible acceder a la ruta que está tratando la petición actual:

```php
$route = Route::current();  // ruta actual
$action = Route::currentRouteAction();  // acción de la ruta actual
```

## Middleware

Se trata de componentes por los que pasa la petición *HTTP* antes de ser atendida. Por ejemplo, el *middleware* que verifica si el usuario está autenticado, redirigirá el *request* hacia la página solicitada o hacia la página de *login*.

El *middleware* se ubica en ***app/Http/Middleware***. Para crear uno nuevo:

```
php artisan make:middleware NombreMiddleware
```

***[TO-DO]***

## Controladores

Para no tener que definir el manejo de los *requests* como *closures* en los archivos de rutas, se pueden usar controladores, que son clases que permiten agrupar lógica de tratamiento de peticiones. Los controladores están ubicados en ***app/Http/Controllers***.

Un controlador puede extender la clase ***App\Http\Controller*** si desea tener acceso a determinadas características. De lo contrario, no es obligatorio.

Si queremos asignar una ruta a un método específico de un controlador:

```php
Route::get('foo', 'NombreController@nombreMetodo');
```

No es necesario especificar la ruta (*namespace*) completa del controlador, porque *Laravel* ya sabe que los controladores están en ***App\Http\Controllers***. Sin embargo, si están en *namespace* interior, sí hay que definirlo. Por ejemplo, si queremos asignar una ruta al método `show()` del controlador ***App\Http\Controllers\Photos\AdminController***:

```php
Route::get('foo', 'Photos\AdminController@show');
```

Por convenio, el nombre de una clase de controlador termina en "Controller".

Si el controlador **únicamente** maneja una sola acción, se puede definir un método `__invoke()`, que es el que será llamado cuando al definir la ruta no se indica nombre de método.

Para crear un controlador:

```
php artisan make:controller nombreController
```

Si queremos que sea invocable (con método `__invoke()`), añadimos `--invokable`.

Cuando enrutamos a un método de controlador, si este requiere parámetros, se añaden en un *array*:

```php
Route::get('foo', 'NombreController@nombreMetodo', ['parm1' => 3, 'parm2' => 'hello']);
```

## Peticiones (*requests*)

Se puede acceder a la *request* actual a través de la inyección de dependencias al hacer un *type-hint* de la clase ***Illuminate\Http\Request***. El *service container*, en este caso, nos enviará una instancia de la petición entrante.

El *service container* también enviará la instancia de la petición en caso de hacer el *type-hint* en una *closure*.

Si por ejemplo nuestro método controlador espera entrada de un parámetro de la ruta, se deberá incluir ese parámetro después de los parámetros correspondientes a las dependencias. Por ejemplo, si definimos la ruta:

```php
Route::get('user/{id}', 'UserController@user');
```

Entonces, suponiendo que nuestro controlador quiera acceder a la *request* actual y al parámetro de la *URL*:

```php
namespace App\Html\Controllers;

use Illuminate\Http\Request;

class UserController extends Controller
{
    public function user(Request $req, $identidad) { /* ... */ }
}
```

### Métodos de la *request* actual

El método `path()` retorna la *URI*.

El método `is()` comprueba si la *URI* coincide con cierto patrón que se le pasa como argumento (permite *wildcards*).

`url()` retorna la *URL* completa, pero sin la *query string*, mientras que `fullUrl()` la incluye.

Para obtener el verbo (método) *HTTP*, `method()` retorna este. También existe `isMethod()`:

```php
if($req->isMethod('post'))
    /* ... */
```

Con el método `all()` se pueden obtener todos los datos de entrada, el cual devuelve un *array*.

El método `input()` retorna un dato de entrada independientemente del verbo *HTTP* usado:

```php
$valor1 = $req->input('name');
$valor2 = $req->input('age', 20);
```

En el segundo caso, se le ha dado un valor por defecto (que retorna cuando no encuentra ese dato).

Si el formulario contiene *arrays*, se usa la notación con punto (***.***) para acceder a estos:

```php
$valor = $req->input('products.0.name');
$valores = $req->input('products.*.name');
```

Si no indicamos ningún argumento, retornará un *array* con todos los datos.

Si solo deseamos los datos del *query string*, se usa el método `query()`, cuyo uso es igual al de `input()`.

Se puede usar una propiedad dinámica de ***Request*** para acceder a los datos:

```php
$valor = $req->name;
```

Primero buscará un dato de entrada con ese nombre, y si no lo encuentra, lo buscará en los parámetros de la ruta, si los hay.

Si queremos obtener datos *json* de una *json request*, usaremos el método `input()`. Para acceder al contenido se usará la notación con punto.

Si queremos leer un valor como booleano, el método `boolean()` retorna ***true*** si el contenido del dato es 1, "1", true, "true", "on" o "yes". En los demás casos retornará ***false***:

```php
$archived = $req->boolean('archived');
```

Si deseamos obtener solo un subconjunto de los datos de entrada, podemos usar los métodos `only()` o `except()`, a los que pasaremos la lista de los datos que deseamos obtener, ya sea en una lista de argumentos, o en un *array* con todos ellos.

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

El método `filled()` es como `has()` con un argumento no *array*, pero retornará ***false*** en el caso que el dato exista pero esté vacío.

El método `missing()` es el inverso de `has()`.

Para obtener el valor de una *cookie* (en *Laravel* están encriptadas y firmadas):

```php
$valor = $req->cookie('name');
```

Otra manera es mediante la *facade* ***Cookie***:

```php
$valor = Cookie::get('name');
```

Métodos relacionados con archivos subidos:

```php
if($req->hasFile('photo'))
    $archivo = $req->file('photo');
if $archivo->isValid() ...
```

El método `file()` retorna una instancia de ***Illuminate\Http\UploadedFile***. A parte de `isValid()`, esta clase dispone de los métodos `path()` (la ruta completa del archivo) o `extension()` (extensión del archivo basada, no en la extensión proporcionada, sino en el contenido).

## Respuestas (*responses*)

La forma más simple de retornar una respuesta (desde una ruta o controlador) es retornando un simple *string*. También podemos retornar un *array*, que el *framework* convertirá a una respuesta *JSON*.

Aunque normalmente se retornará una vista o una ***Illuminate\Http\Response***. En este último caso podemos personalizar, además, el código de respuesta y las cabeceras *HTTP*:

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

También se pueden añadir *cookies* a la respuesta. Se hace de la misma forma que el método `header()`, pero en este caso con el método `cookie()`, cuyos tres primeros parámetros son los más usados: nombre, valor y tiempo de vida (en minutos). Alternativamente se puede usar el *facade* ***Cookie*** para añadir *cookies* a la cola, que serán enviadas con la respuesta (mismos argumentos que opción anterior):

```php
Cookie::queue(Cookie::make($nombre, $valor, 120));
Cookie::queue($nombre, $valor, 120);    // equivalente
```

Por defecto, las *cookies* están encriptadas y firmadas (no pueden ser modificadas por el cliente).

Para generar una redirección como respuesta:

```php
Route::get('dashboard', function() {
    return redirect('home/dashboard');
});
```

Si hay que redirigir el usuario a su localización anterior, por ejemplo, si ha enviado un formulario inválido y hay que retroceder conservando los datos que ha tecleado:

```php
return back();  // aquí se perderían los datos tecleados
return back()->withInput();    // aquí se conservan
```

Para redirigir a la acción de un controlador:

```php
return redirect()->action('NombreController@metodo');
```

Se puede pasar un segundo argumento con un *array* de parámetros.

Para redirigir a un sitio externo:

```php
return redirect()->away('https://www.somedomain.com');
```

Si además de retornar una respuesta personalizada como las que hemos visto, el contenido debe ser una vista, hay que añadir también el método `view()`:

```php
return response($contenido)
            ->header('Content-type', $type)
            ->view('vistaHello', $data, 200);
```

En caso de que no queramos personalizar cabeceras, y simplemente queramos retornar una vista, podemos simplemente usar la función `view()`:

```php
return view('vistaHello', $data, 200);
```

Para una respuesta *JSON*, se puede usar el método `json()`, el cual establecerá automáticamente la cabecera ***'Content-Type'*** a ***'application/json'***, y convertirá el *array* proporcionado a formato *JSON* usando la función de *PHP* `json_encode()`:

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

Contienen el *HTML* servido, y separa los controladores (lógica de la aplicación) de la presentación.

Se almacenan en ***resources/Views***.

Cuando definimos una ruta así:

```php
Route::get('/', function() {
    return view('admin.greeting', ['name' => 'James', 'age' => 55]);
});
```

Estamos retornando el contenido de la plantilla *Blade* ***resources/Views/admin/greeting.blade.php***, a la que le pasamos dos argumentos. Obsérvese la *dot notation* en la llamada a `view()`: los subdirectorios de vistas no deben contener el carácter punto (***.***).

Podemos comprobar si existe una vista a través de la *facade* ***View***:

```php
use Illuminate\Support\Facades\View;

if(View::exists('email.customer.form')) ...
```

Con el método `first()` se retorna la primera vista disponible de una lista:

```php
return view()->first(['email.customer', 'customer', 'general.customer'], $args);
```

El *array* de argumentos es opcional.

Los argumentos se pueden pasar a una vista mediante ese *array* en el segundo argumento de `view()`, o mediante el método `with()`:

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

Se trata de código (*callback* o método) invocado cuando se renderiza una vista. Para **registrar** un *view composer* (normalmente mediante un *service provider*), se utiliza el método `composer()` de la *façade* ***View***:

```php
View::composer('greeting', 'App\Http\View\Composers\GreetingComposer');
```

En este caso se ha dado la ruta completa de *namespaces* del *composer*, puesto que en *Laravel* no tienen una ubicación específica. Se pueden organizar en cualquier parte de la jerarquía que nos interese. Por otro lado, en lugar del nombre de una clase, se puede dar como segundo argumento una *closure*.

En el ejemplo, **justo antes** de renderizar la vista ***greeting***, se llamará al método `compose()` de la clase especificada, que recibe como parámetro una instancia de dicha vista (de la clase ***Illuminate\View\View***). En el código del método podemos manipular este objeto, por ejemplo añadiéndole datos:

```php
public function compose(View $view)
{
    $view->with('count', 50);
}
```

Si se quiere registrar un *view composer* para más de una vista, en lugar del nombre de la vista, se puede usar un *array* con los nombres de las vistas deseadas. A parte, se permite el uso de *wildcards*, con lo que, por ejemplo, asociar un *view composer* a ***'*'*** lo asociará a todas las vistas de la aplicación.

### Optimización

Las vistas se compilan y guardan para usos futuros. La versión compilada tiene preferencia cuando es posible presentarla (si la plantilla ha cambiado se vuelve a compilar). Pero estas compilaciones ocurren *on demand*, lo cual afecta negativamente al desempeño. Se puede ejecutar en línea de comandos:

```
php artisan view:cache
```

Esto compilará todas las vistas. Si se quiere borrar todo el caché de vistas:

```
php artisan view:clear
```

## Generación de *URLs*

Dada una *URI*, la función `url()` construye una *URL* completa, añadiendo a esa *URI* el esquema (*http* o *https*) y el nombre del *host*, correspondientes a la *request* actual. Si no se le pasa una *URI*, retornará un **objeto** con la *URL* actual. A partir de este, podemos acceder a la siguiente información:

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

Podemos obtener una *URL* para un método concreto de un controlador con la función `action()`:

```php
$url = action('NombreController@metodoX');
$url = action([NombreController::class, 'metodoX']);
```

Cualquiera de las dos formas aceptará también un segundo argumento con un *array* de argumentos al método.

## Sesión

La configuración de sesión está en ***config/session.php***. Por defecto, *Laravel* utiliza el *driver* de sesión ***file*** (está establecido en la opción de configuración ***driver***). Los posibles *drivers* son:

- ***file*** - sesiones almacenadas en ***storage/framework/sessions***.
- ***cookie*** - se almacenan en *cookies* encriptadas.
- ***database*** - almacenadas en una base de datos relacional.
- ***memcached***, ***redis*** - almacenadas en cachés en memoria rápidos.
- ***array*** - en un *array PHP* (sin persistencia).

### Uso de la sesión

Se puede acceder a los datos de sesión mediante la función `session()` o mediante una instancia de ***Request***, que puede ser *type-hinted* en un controlador. En este último caso, el acceso es de este tipo:

```php
public function show(Request $req)
$valor = $req->session()->get('key');
```

A este método `get()` se le puede pasar un segundo argumento opcional, que será devuelto en caso de que la clave solicitada no exista. Este argumento opcional puede ser una *closure*, que será ejecutada y se usará el valor que retorne.

También se puede usar la función global `session()`, cuyo uso es igual, pero en este caso podemos establecer valores de sesión pasándole como argumento un *array* con pares clave/valor.

Para obtener todos los datos de sesión en un *array*:

```php
$data = $req->session()->all();
```

Para determinar si un determinado dato existe:


```php
if($req->session()->has('key')) ...
```
Retorna ***true*** si el elemento existe y no es ***null***. Similar a:

```php
if($req->session()->exists('key')) ...
```

En este caso, si el elemento existe, el método retornará ***true***, aunque su valor sea ***null***.

Para almacenar un dato:

```php
$req->session()->put('clave', 'valor');
session(['clave' => 'valor']);    // equivalente
```

Si la clave es un *array*, debe hacerse con `push()` en lugar de con `put()`:

```php
$req->session()->push('user.teams', 'developers');
```

Esto añade un valor a la clave ***user.teams***, que contiene un *array*.

Un método equivalente a `get()` es `pull()`, con la diferencia que este último, si la clave existe, la borra después de leerla.

Para incorporar datos que solo estén disponibles en la siguiente petición *HTTP* (por ejemplo, un mensaje de confirmación del tipo 'consulta enviada'), se hace mediante el método `flash()`.

```php
$req->session()->flash('status', 'Mensaje enviado con éxito');
```

En este caso, el elemento está disponible durante la *request* actual y durante la siguiente únicamente. Si en la siguiente solicitud decidimos ampliar la disponibilidad de los datos *flashed* una *request* más:

```php
$req->session()->reflash();
```

Y si solo deseamos hacerlo con un subconjunto de los datos *flashed*:

```php
$req->session()->keep(['clave1', 'clave2', 'clave3']);
```

Para eliminar datos de la sesión:

```php
$req->session()->forget('key');    // elimina un dato de sesión
$req->session()->forget(['key1', 'key2', 'key3']);    // varios datos
$req->session()->flush();    // todos los datos
```

## Validación

Lo más utilizado es el método `validate()` del objeto ***Request*** actual (podemos acceder a él desde cualquier método mediante inyección).

Al invocar este método, se le deben pasar las reglas de validación en un *array*:

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
]);
```

Se supone que esto se ejecuta cuando la *request* es el *submit* de un formulario, por ejemplo. Si la validación falla, el código posterior a `validate()` en nuestro controlador no se ejecuta, y se retorna al estado anterior al *submit* (se genera una respuesta automáticamente, y volveríamos a ver el formulario en blanco, para volver a introducir los datos). Si la validación es correcta, podemos seguir con el código que tratará los valores recibidos del *submit*.

Las reglas de validción pueden pasarse como un *array* también: `['required', 'unique:posts', 'max:255']`. Ver el manual para los tipos de validación disponibles.

Cuando la validación ha fallado, y se recibe la respuesta, los errores son accesibles a través de la variable `$errors`, que es un *array* con todos ellos. En la plantilla *Blade* se puede comprobar si existe alguno:

```html
@if ($errors->any())
  <ul>
    @foreach ($errors->all() as $error)
      <li>{{ $error }}</li>
    @endforeach
  </ul>
@endif
```

Alternativamente puede usarse la directiva *Blade* `@error`, y la variable ***$message***:

```html
@error('title')
  <p>{{ $message }}</p>
@enderror
```

En caso de que se haya producido un error de validación, normalmente volveremos a cargar el formulario, pero se perderán los valores que hemos enviado (al usar `validate()`, si esta falla, ya no tendremos disponible los valores de entrada del formulario). Para ello, se podrá usar el valor de la entrada antigua mediante la función helper `old()`, que recoge la entrada antigua. Se puede utilizar en la plantilla *Blade*:

```html
<input type="text" name="DNI" id="dniform" value="{{ old('DNI') }}">
```

Por ejemplo, suponiendo un formulario que tenga la línea anterior en un formulario:

```php
public function creacion() {  // responde a GET (presenta el formulario vacío)
    return view('formulario');
}

public function guardado(Request $req) {  // responde al submit POST, recibe inyección de la request
    $req->validate([
        'DNI' => 'min:8|max:9|required',
        'Nombre' => 'required'
    ]);
    return view('formulario');
}
```

Si la validación falla, `validate()` terminará el método retornando el anterior formulario con los datos de `old()` disponibles para ese formulario, es decir, se recargará el formulario con las entradas anteriores (de hecho retorna `back()->withInputs()`). En cambio, si tiene éxito la validación, se recargará el formulario nuevo, sin datos en `old()` vacíos (normalmente, para poder usar los campos antiguos con `old()` en la siguiente *request*, hay que *flash* los datos en la *request* actual, mediante el método `flash()` de la *request*).

Si por ejemplo deseamos un valor por defecto (por ejemplo una ciudad de residencia), cuando no hay valor antiguo que corregir, se puede hacer así:

```html
<input type="text" name="ciudad" id="ciudadform" value="{{ old('DNI') ?? 'Sabadell' }}">
```

Así, en el caso de que el valor antiguo exista, retornará este; de lo contrario, será 'Sabadell'. Este es el uso del operador `??` de *Blade* (similar al operador ternario).

### Reglas de validación personalizadas

Si deseamos añadir una regla de validación (*custom rule*), debemos crear una regla:

```
php artisan make:rule NombreRegla
```

La regla queda guardada en ***app/Rules***. La regla tendrá los métodos `passes()` y `message()`. El primero recibe el nombre del atributo y su valor (2 parámetros), y debe retornar ***true*** o ***false*** según pase la validación o no. Por su parte, el método `message()` debe retornar el mensaje de error en caso de que falle la validación.

Para utilizar la regla, simplemente se debe pasar una instancia a la lista de validaciones para un atributo:

```php
$req->validate([
    'DNI' => ['min:8', 'max:9', 'required', new NombreRegla],
    'Nombre' => 'required'
]);
```