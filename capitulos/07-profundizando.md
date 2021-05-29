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

### URL's

#### route()

El *helper* `route()` retorna la *URL* correspondiente a una ruta **con nombre**. Por ejemplo, `route('nombreRuta')` retornaría la *URL* correspondiente a la ruta que tiene como nombre ***nombreRuta***. Si se define una ruta que contiene parámetros, se deben pasar tantos argumentos como sean necesarios a este *helper* mediante un array. Por ejemplo, si hemos definido la ruta:

```php
Route::get('clients/{id}', /* closure o controller */) -> name('clientesForm');
```

Podemos obtener la *URL* así: `route('clientesForm', ['id' => 33])`.

Las *URLs* serán absolutas. Si deseamos una *URL* relativa, debemos pasar un tercer argumento con el valor ***false***.

#### asset()

Este *helper* retornará la *URL* de un *asset*. Dado que la *URL* tiene como base, por defecto, el raíz del proyecto (el directorio donde se halla ***index.php***), es decir, el directorio ***public***. Por lo tanto, todos los *assets* que necesitemos (imágenes, *scripts .js*, etc.), deberían estar en ese directorio o en subcarpetas del mismo. Si por ejemplo tenemos imágenes en ***public/images***, la *URL* de una de esas imágenes podría ser `asset('images/logo.png')`.

Podemos definir la base de estas *URL* mediante la variable `ASSET_URL` del archivo ***.env***, lo cual es útil si tenemos *assets* en servidores externos:

```
ASSET_URL = https://undominio.com/assets
```

Entonces, `asset('images/logo.png')` retornará la *URL* ***https://undominio.com/assets/images/logo.png***.


## Desarrollo de paquetes

### Publicación de la configuración

Una vez hayamos incluido un paquete en nuestro proyecto con `composer`, necesitamos publicar la configuración del mismo, dentro del árbol de directorios de nuestro proyecto (es decir, fuera del subdirectorio de ***vendor*** donde resida el paquete). Los archivos de configuración que serán publicados están definidos en el *service provider* que viene con el paquete.

Para publicar la configuración de todos los paquetes que no lo hayan hecho:

```
php artisan vendor:publish
```

Para publicar la configuración de un paquete concreto usaremos la opción `--provider`.
