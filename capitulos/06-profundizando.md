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
