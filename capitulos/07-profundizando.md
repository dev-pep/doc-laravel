# Profundizando

## Eventos

Podemos definir eventos a los que escucharemos mediante *listeners*. Las clases de eventos se guardan por defecto en ***app/Events***, mientras que los *listeners* lo hacen en ***app/Listeners***.

Un buen sitio para registrar los *listeners* es en el *service provider* ***EventServiceProvider***, el cual tiene una propiedad ***$listen*** que contiene un *array* cuyas claves son los eventos, y cuyos valores son *arrays* de *listeners* (un evento puede tener más de un *listener*).

```php
protected $listen = [
    'App\Events\OrderShipped' => [
        'App\Listeners\SendShipmentNotification',
    ],
];
```

Una vez definidos estos eventos de este modo, podríamos proceder a crear los correspondientes archivos de eventos y de *listeners*, pero para evitarnos trabajo, se pueden crear de un plumazo con `artisan`:

```
php artisan event:generate
```

Esto generará todos los archivos necesarios (los ya creados se dejarán igual).

A parte de registrar los eventos en el *array*, también se pueden asignar *closures* a eventos concretos dentro del método `boot()` del ***EventServiceProvider***.

```php
public function boot()
{
    parent::boot();
    Event::listen('event.nombre', function ($foo, $bar) { /*...*/  });
}
```

El nombre del evento puede incluso incluir *wildcards*, como ***'event.*'*** para asociar más de un evento a esa closure. De todos modos, todo lo registrado así no será generado automáticamente por `artisan`.

### Autodescubrimiento de eventos

Podemos optar por no registrar absolutamente ningún evento, y permitir a *Laravel* descubrir los eventos y escuchadores automáticamente examinando el directorio ***app/Listeners***. Por defecto, el descubrimiento de eventos está deshabilitado, por lo que hay que habilitar el descubrimiento de eventos *overriding* este método del ***EventServiceProvider***:

```php
public function shouldDiscoverEvents() {
    return true;
}
```

Para descubrir eventos, *Laravel* busca en los escuchadores un método que empiece por ***handle***. Entonces lo asocia al evento del tipo especificado en la lista de parámetros de ese método:

```php
use App\Events\PodcastProcessed;

class SendPodcastProcessedNotification
{
    public function handle(PodcastProcessed $event) { /*...*/  }
}
```

En este caso, el escuchador queda ligado al evento de tipo ***PodcastProcessed***.

De todas formas, si también hay eventos en el *array* ***$listen***, también se registrarán.

### Definición y despacho de eventos

Para despachar (generar) un evento se puede usar el *helper* `event()` (desde cualquier parte del código) al que se le pasará una instancia del evento deseado, conteniendo la información que queramos. Para ello, el constructor de la clase evento almacenará adecuadamente la información que reciba su constructor.

Al despacharse el evento, los escuchadores asociados recibirán esa instancia a través de su método *handler*, y lo procesarán adecuadamente.

## *Helpers*

A parte de numerosas utilidades para *arrays*, *strings* y otras cosas, podríamos destacar estos *helpers*:

### *Paths*

Es posible obtener la ruta (absoluta) a directorios concretos del proyecto mediante las siguientes funciones:

- `app_path()` es la ruta a la carpeta ***app***.
- `base_path()` es la ruta a la raíz del proyecto.
- `config_path()` es la ruta a la carpeta ***config***.
- `database_path()` es la ruta a la carpeta ***database***.
- `public_path()` es la ruta a la carpeta ***public***.
- `resource_path()` es la ruta a la carpeta ***resources***.
- `storage_path()` es la ruta a la carpeta ***storage***.

Si opcionalmente les pasamos un argumento a cualquiera de estas funciones, con un *string* que indique una ruta (relativa al directorio en cuestión) a un archivo o directorio, retornará la ruta (absoluta) de ese archivo o directorio.

### URL's

#### asset() y secure_asset()

Este *helper* retornará la *URL* de un *asset*. Dado que la *URL* tiene como base, por defecto, el raíz del proyecto (el directorio donde se halla ***index.php***), es decir, el directorio ***public***. Por lo tanto, todos los *assets* que necesitemos (imágenes, *scripts .js*, etc.), deberían estar en ese directorio o en subcarpetas del mismo. Si por ejemplo tenemos imágenes en ***public/images***, la *URL* de una de esas imágenes podría ser `asset('images/logo.png')`.

Podemos definir la base de estas *URL* mediante la variable `ASSET_URL` del archivo ***.env***, lo cual es útil si tenemos *assets* en servidores externos:

```
ASSET_URL = https://undominio.com/assets
```

Entonces, `asset('images/logo.png')` retornará la *URL* ***https://undominio.com/assets/images/logo.png***.

La función `secure_asset()` es igual, pero retorna una *URL HTTPS*.

#### route()

El *helper* `route()` retorna la *URL* correspondiente a una ruta **con nombre**. Por ejemplo, `route('nombreRuta')` retornaría la *URL* correspondiente a la ruta que tiene como nombre ***nombreRuta***. Si se define una ruta que contiene parámetros, se deben pasar tantos argumentos como sean necesarios a este *helper* mediante un array. Por ejemplo, si hemos definido la ruta:

```php
Route::get('clients/{id}', /* closure o controller */) -> name('clientesForm');
```

Podemos obtener la *URL* así: `route('clientesForm', ['id' => 33])`.

Las *URLs* serán absolutas. Si deseamos una *URL* relativa, debemos pasar un tercer argumento con el valor ***false***.

#### url() y secure_url()

El *helper* `url()` retorna la *URL* de un archivo o carpeta del proyecto que se le pasa como primer argumento. Si se invoca sin argumentos, retorna un objeto ***UrlGenerator***, que acepta métodos como `current()`, `full()` o `previous()`.

La función acepta un segundo argumento con un *array* con componentes que se añaden a la *URL*:

```
url('user/profile', [532])
```

La función `secure_url()` es igual, pero retorna una *URL HTTPS*.

### Varios

#### back()

La función `back()` retorna una *response* de redirección a la localización anterior.

```php
return back($status = 302, $headers = [], $fallback = false);
return back();
```

Supongamos que nos presentan un formulario para introducir datos. Esta es la localización original (fruto, de hecho, de una *response* de la aplicación). Rellenamos los campos y realizamos una *request* que envía esos datos. Un controlador decide que los datos son incorrectos, con lo que tenemos que volver al formulario, es decir, tenemos que volver atrás, nuevamente a la *response* original, no retornar una nueva. Ahí es cuando usamos `back()`. Pero, si además queremos que esa respuesta venga rellena con los datos que el usuario ha escrito (aunque sean incorrectos, es mejor que no pierda lo que ha escrito), lo haremos así:

```php
return back() -> withInput();
```

Véase también `old()`.

#### config()

Con el *helper* `config()` obtenemos valores de configuración. Utiliza notación de puntos para avanzar por la jerarquía de directorios (directorio ***config***) y por la jerarquía de *arrays* y objetos del archivo de configuración. Acepta un segundo argumento opcional, con un valor por defecto.

```php
$valor = config('app.timezone', $default);
```

Para **escribir** un valor de configuración se utiliza un *array*:

```php
config(['app.timezone' => $timz]);
```

#### cookie()

Crea una instancia de una *cookie* con el nombre, valor y número de minutos de expiración indicados.

```php
$cookie = cookie('La cookie', 'Este es el valor', 180);
```

#### env()

Retorna el valor de una variable del sistema (o variable definida en archivo ***.env***). Acepta un segundo argumento con valor por defecto.

```php
$valor = env('REMOTE_USER', 'Pepito');
```

#### old()

Antes de retornar la respuesta, podemos *flashear* en la sesión actual la entrada del usuario para que esté disponible en la siguiente *request*. Lo haremos así:

```php
$request -> flash();
```

Podemos *flash* solo parte de la entrada:

```php
$request->flashOnly(['username', 'email']);
$request->flashExcept('password');
```

Ahora, tras retornar la respuesta, en la siguiente *request* podemos acceder a estos datos con `old()`:

```php
$antiguo_nombre = old('nombre');
```

Podemos pasarle un segundo argumento con un valor por defecto en caso de que no encuentre el dato.

Si lo que queremos es redirigir a un formulario con datos incorrectos, de tal modo que esos datos vuelvan a mostrarse en el formulario, deberemos utilizar el valor antiguo en la vista. En la plantilla *Blade* usaremos como valor por defecto del *form input* pertinente el valor retornado por `old()`. Por ejemplo:

```html
<input type="text" id="ciudad" name="ciudad" value="{{ old('ciudad') }}">
```

Si no hay tal dato antiguo, `old()` retornará un valor nulo, con lo que el campo aparecerá en blanco, correctamente. Si queremos dar un valor por defecto cuando no exista dicho campo por defecto, podemos usar el operador `??`:

```html
<input type="text" id="ciudad" name="ciudad" value="{{ old('ciudad') ?? 'Beijing' }}">
```

#### redirect()

El *helper* `redirect()` retorna una *response* de redirección. Sin argumentos, retorna una instancia del redirector.

```php
return redirect($to = null, $status = 302, $headers = [], $secure = null);

return redirect('/home');

return redirect()->route('route.name');
```

#### request()

La función `request()` retorna una instancia de la *request* actual. También podemos usarla para obtener un valor de entrada (acepta un segundo argumento con valor por defecto):

```php
$request = request();
$valor = request('clave', $default);
```

#### response()

El *helper* `response()` retorna una instancia de una respuesta del servidor (ver sección de respuestas).

## Desarrollo de paquetes

### Publicación de la configuración

Una vez hayamos incluido un paquete en nuestro proyecto con `composer`, necesitamos publicar la configuración del mismo, dentro del árbol de directorios de nuestro proyecto (es decir, fuera del subdirectorio de ***vendor*** donde resida el paquete). Los archivos de configuración que serán publicados están definidos en el *service provider* que viene con el paquete.

Para publicar la configuración de todos los paquetes que no lo hayan hecho:

```
php artisan vendor:publish
```

Para publicar la configuración de un paquete concreto usaremos la opción `--provider`.
