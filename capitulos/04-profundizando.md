# Profundizando

## Colecciones (*collections*)

La clase ***Illuminate\\Support\\Collection*** proporciona un potente mecanismo para el tratamiento de *arrays* de datos.

Para crear una *collection* solo hay que ejecutar el *helper* `collect()`, pasándole como argumento un *array* y recoger la colección retornada.

Las colecciones disponen de numerosos métodos para el tratamiento de los datos. Véase la documentación oficial para una descripción exhaustiva de estos.

## Eventos

Podemos definir eventos a los que estaremos atentos (escucharemos) mediante *listeners*. Las clases de eventos se guardan por defecto en ***app/Events***, mientras que los *listeners* lo hacen en ***app/Listeners***.

Un buen sitio para registrar los *listeners* es en el *service provider* ***EventServiceProvider***, incluido por defecto en *Laravel*, el cual tiene una propiedad ***\$listen*** que contiene un *array* cuyas claves son las clases *fully qualified* de los diferentes eventos, y los valores son *arrays* que contienen las clases de *listeners* asociados. Un evento puede ser escuchado por más de un *listener*, aunque un *listener* no puede escuchar más de un evento.

```php
protected $listen = [
    MiEvento1::class => [
        MiListener1::class,
        MiListener2::class
    ],
    MiEvento2::class => [ MiListener3::class ]
];
```

Para ver todos los eventos y *listeners* registrados en la aplicación:

```
php artisan event:list
```

Una vez registrados los eventos y *listeners*, podríamos proceder a crear los correspondientes archivos *PHP* con las clases para los eventos y los *listeners*, pero para evitarnos trabajo, se pueden crear de un plumazo con `artisan`:

```
php artisan event:generate
```

Esto creará y todos los eventos y *listeners* que estén registrados en el proveedor ***EventServiceProvider*** (los archivos ya creados se dejarán igual).

Alternativamente, se pueden crear los eventos y *listeners* mediante el comando `make` de *artisan*:

```
php artisan make:event MiEvento
php artisan make:listener MiListener --event=MiEvento
```

En este caso, no se registra ni el evento ni el *listener* en ningún *service provider*. Lo que hace el *flag* `--event` es añadir el tipo de evento en la lista de parámetros del método *handler* del *listener*. Si no se incluye este *flag*, el parámetro se indica sin tipo (véase más adelante).

A parte de registrar los eventos en el *array* ***\$listen*** del *service provider*, también se pueden registrar clases de *listeners* y/o *closures* a eventos concretos dentro del método `boot()` del ***EventServiceProvider***.

```php
use App\Events\MiEvento;
use App\Listeners\MiListener;
use Illuminate\Support\Facades\Event;

public function boot()
{
    Event::listen(
        MiEvento::class,
        [MiListener::class, 'handle']
    );
Event::listen(function(MiEvento $evento) { /* ... */ });
}
```

Nótese el uso de la *facade* ***Illuminate\Support\Facades\Event***. El primer registro asocia el evento ***MiEvento*** al método `handle()` del *listener* ***MiListener***, mientras que el segundo asocia el mismo evento también a la *closure* definida, que recibe una instancia de dicho evento.

### Autodescubrimiento de eventos

Podemos optar por no registrar absolutamente ningún evento, y permitir a *Laravel* descubrir los eventos y *listeners* automáticamente examinando el directorio ***app/Listeners***. Por defecto, el descubrimiento de eventos está deshabilitado, por lo que hay que habilitarlo *overriding* este método del ***EventServiceProvider***:

```php
public function shouldDiscoverEvents() {
    return true;
}
```

Para descubrir eventos, *Laravel* busca dentro de los *listeners* de la carpeta ***app/Listeners*** un método que empiece por ***handle*** o por ***__invoke***. Entonces lo asocia al evento del tipo especificado su la lista de parámetros:

```php
class MiListener
{
    public function handle(MiEvento $event) { /*...*/  }
}
```

En este caso, el *listener* ***MiListener*** queda escuchando al evento ***MiEvento***.

De todas formas, se puede combinar el autodescubrimiento con el registro de eventos en el *array* ***\$listen***.

### Definición de eventos

Un evento es un simple contenedor de datos (los datos del evento producido), que serán establecidos en el momento de despachar el evento. Por lo tanto, lo habitual es que posea un simple constructor que almacene los datos recibidos.

### Definición de *listeners*

Pueden tener un constructor, al que se puede inyectar cualquier servicio necesario. Por otro lado reciben, como único parámetro de su método *handler*, una instancia del evento generado. El método *handler* por defecto es `handle()` cuando creamos el *listener* mediante *artisan*.

Es importante tener en cuenta que si el método *handler* retorna ***false***, el evento no se propagará al resto de *listeners* que estén escuchando el evento y no se hayan ejecutado todavía.

### Despacho de eventos

Para despachar (generar) un evento se puede usar el método estático `dispatch()` de la clase del evento concreto. Los argumentos que pasemos a este método serán recibidos por el constructor del evento.

### Suscriptores

Una clase suscriptora es similar a un *listener*, con la diferencia que puede suscribirse a más de un evento. Para ello debe definir, para cada evento, un método *handler* distinto.

En la misma definición de la clase se puede indicar a qué evento corresponde cada método *handler*, mediante el método `subscribe()`, que retornará un *array* con dicha información.

```php
namespace App\Listeners;

use App\Events\EventoUno;
use App\Events\EventoDos;

class MiSuscriptor
{
    public function handleEventoUno($evento) {
        /* ... */
    }
    public function handleEventoDos($evento) {
        /* ... */
    }
    public function subscribe() {
        return [
            EventoUno::class => 'handleEventoUno',
            EventoDos::class => 'handleEventoDos'
        ]
    }
}
```

Una vez hecho esto, se registrará el suscriptor, por ejemplo, en ***EventServiceProvider***, en la propiedad ***\$subscribe***:

```php
protected $subscribe = [
    MiSuscriptor::class,
    /* ... */
]
```

## *File storage*

Si queremos permitir la subida de archivos a la aplicación, hay que tener en cuenta que el formulario de subida debe ser de tipo ***POST***, y que la etiqueta `<form>` debe contener el atributo *HTML* `enctype="multipart/form-data"`.

Por otro lado, *PHP* debe estar configurado para permitir las subidas de archivos, con lo que el archivo ***php.ini*** deberá contener la línea `file_uploads=On`.

Finalmente, incluiremos nuestros campos *HTML* `<input type="file" ...>`.

En *Laravel*, la configuración de archivos se encuentra en ***config/filesystems.php***, en el que podemos definir nuestros "discos", caracterizados por un *driver* (admitidos ***local***, ***ftp***, ***sftp*** y ***s3***) y una ubicación concreta.

El *driver* ***local*** es el único que no precisa de la instalación de paquetes extra, y nos sirve para almacenar los archivos en las carpetas locales del proyecto. Por convenio y claridad, se suele usar el directorio ***storage*** y sus subcarpetas.

El archivo de configuración también define el disco por defecto.

Si queremos definir un disco público, de tal modo que los archivos allí subidos sean accesibles públicamente (a través de su *URL*), usaremos el disco ya preconfigurado por defecto llamado ***public*** (podemos editarlo o cambiarle el nombre). Utiliza el *driver* local, y por defecto está asociado a ***storage/app/public***. Para que se pueda acceder a través de *URL* a sus archivos hay que crear un enlace simbólico en ***public***. Esto se puede hacer fácilmente con el comando:

```
php artisan storage:link
```

Así, se creará el enlace simbólico ***public/storage***, que enlaza con ***storage/app/public***. A partir de entonces ya podemos acceder a los archivos mediante sentencias del tipo `asset('storage/archivo.txt')`.

Este enlace queda definido en el archivo de configuración:

```php
'links' => [
    public_path('storage') => storage_path('app/public')
]
```

Así, podemos definir otros enlaces simbólicos, bien editando ***filesystems.php*** o bien usando `artisan`, que cambiará ese archivo de configuración adecuadamente.

### Requisitos

Algunos *drivers* tienen unos prerequisitos que de deben cumplirse. Por ejemplo, el *driver* ***FTP*** necesita el paquete ***flysystem-ftp***:

```
composer require league/flysystem-ftp "^3.0"
```

Además, habrá que configurarlo en ***config/filesystems.php***:

```php
'ftp' => [
    'driver' => 'ftp',
    'host' => env('FTP_HOST'),
    'username' => env('FTP_USERNAME'),
    'password' => env('FTP_PASSWORD'),

    // Opcional:
    // 'port' => env('FTP_PORT', 21),
    // 'root' => env('FTP_ROOT'),
    // 'passive' => true,
    // 'ssl' => true,
    // 'timeout' => 30,
]
```

De forma similar, para ***SFTP***:

```
composer require league/flysystem-sftp-v3 "^3.0"
```

Configuración:

```php
'sftp' => [
    'driver' => 'sftp',
    'host' => env('SFTP_HOST'),

    // Autenticación básica:
    'username' => env('SFTP_USERNAME'),
    'password' => env('SFTP_PASSWORD'),

    // Autenticación mediante clave SSH con password:
    'privateKey' => env('SFTP_PRIVATE_KEY'),
    'password' => env('SFTP_PASSWORD'),

    // Opcional:
    // 'port' => env('SFTP_PORT', 22),
    // 'root' => env('SFTP_ROOT', ''),
    // 'timeout' => 30,
],
```

### Operaciones con los discos

Las operaciones de disco se realizan mediante la *facade* ***Illuminate\\Support\\Facades\\Storage***. Todos los *paths* son relativos a la raíz de nuestro disco definido en la configuración.

Por ejemplo, suponiendo que la variable ***\$filecontents*** almacene el contenido de un archivo ***foto.png***, para guardarlo en la carpeta ***avatars*** de nuestro disco llamado ***local***:

```php
Storage::disk('local')->put('avatars/foto.png', $filecontents);
```

Usando esta *facade*, si no se indica el disco, es como indicar el disco por defecto:

```php
Storage::put('avatars/foto.png', $filecontents);
```

Es posible crear un disco sobre la marcha:

```php
$disk = Storage::build([
    'driver' => 'local',
    'root' => '/path/to/root',
]);
$disk->put('image.jpg', $content);
```

Para obtener el contenido *raw* de un archivo:

```php
Storage::get('textos/texto5.txt');
```

Para saber si un archivo existe o no existe:

```php
if(Storage::disk('disco1')->exists('texto.txt'))
    /* ... */
if(Storage::disk('disco5')->missing('texto.txt'))
    /* ... */
```

### Descarga y otros

Podemos generar una *response* que obligue al cliente a descargar un archivo concreto:

```php
return Storage::download('catalogo.pdf');
```

El método `download()` admite un segundo argumento, opcional, con el nombre con el que aparecerá el archivo al cliente. Un tercer argumento, también opcional, será un *array* con cabeceras *HTTP*.

Para obtener la *URL* de un archivo se usa el método `url()` de ***Storage***, que en *driver* local retorna una *URL* relativa al archivo. En driver ***s3*** (*Amazon cloud storage*) retorna una *URL* absoluta. El método recibe la ruta del archivo.

Existen otros métodos para obtener información del archivo en disco, como `size()` o `lastModified()`.

El método `path()` retorna la ruta absoluta del archivo (en disco o en la nube).

### Almacenamiento

Se pueden almacenar archivos, como hemos visto, con el método `put()`, especificando el directorio, y el contenido. Dicho contenido puede ser el contenido en bruto, o un recurso *PHP*.

También pueden usarse los métodos `prepend()` y `append()` para añadir al principio y al final del archivo respectivamente. Al igual que `put()`, estos métodos reciben la ruta del archivo (relativa al raíz del disco) y el contenido a escribir.

Para copiar y mover archivos, existen los métodos `copy()` y `move()`, los cuales reciben como argumentos la ruta del archivo origen y la ruta destino.

### Subida

Para acceder a uno de los archivos enviados en la *request* actual:

```php
$archivo = $request->file('docu');
```
En este caso, el archivo se ha subido en un formulario donde el *input* ***docu*** se correspondía a un archivo. Para almacenar ese archivo:

```php
$ruta = $request->file('docu')->store('archivos');
```

Lo cual guardará ese archivo en la carpeta ***archivos*** del disco por defecto (no se especifica nombre de archivo). El archivo recibido se guardará con un **nombre único en disco**, y una extensión basada en el tipo *MIME* del archivo (basado en el contenido). El método retornará el *path* completo con el nombre resultante en disco.

La sentencia anterior es equivalente a:

```php
$ruta = Storage::putFile('archivos', $request->file('docu'));
```

Si queremos que guarde el archivo con un nombre concreto:

```php
$ruta = $request->file('docu')->storeAs('archivos', 'archivo55.txt');
$ruta = Storage::putFileAs('archivos', $request->file('docu'), 'archivo55.txt');
```

Las dos sentencias son equivalentes.

Si queremos especificar un disco concreto, en el caso de la *facade* ***Storage*** se usa el método `disk()`, como hemos visto. En el caso de usar `store()`, el nombre del disco se incluirá en un segundo argumento del método. En el caso de `storeAs()`, en un tercer argumento.

### Información de archivos

En el caso de archivos subidos en la *request*, podemos obtener otra información de los mismos:

```php
$size = $request->file('docu')->getSize();  // tamaño (bytes)
$tipo = $request->file('docu')->getClientOriginalName();  // nombre original
$tipo = $request->file('docu')->getClientOriginalExtension();  // extensión original
$tipo = $request->file('docu')->hashName();  // nombre generado
$tipo = $request -> file('docu') -> extension();  // extensión generada (según contenido)
```

Para saber si la *request* actual contiene un archivo concreto:

```php
if($request -> hasFile('docu'))
    /* ... */
```

### Visibilidad

Para almacenar archivos para que sean accesibles públicamente (visibles a otros), hay que marcarlos como públicos. Por ejemplo, pasando un tercer argumento *string* ***public*** a `put()` (de ***Storage***), o usando métodos como `storePublicly()` o `storePubliclyAs()` (del objeto archivo de la *request*).

Para cambiar la visibilidad de archivos ya almacenados:

```php
$visibility = Storage::getVisibility('file.jpg');
Storage::setVisibility('file.jpg', $vis);
```

Aquí, ***$vis*** puede ser un *string* ***public*** o ***private***.

Es posible configurar el significado de público y privado en la configuración del disco *local*:

```php
'local' => [
    'driver' => 'local',
    'root' => storage_path('app'),
    'permissions' => [
        'file' => [
            'public' => 0644,
            'private' => 0600,
        ],
        'dir' => [
            'public' => 0755,
            'private' => 0700,
        ],
    ],
],
```

### Eliminación de archivos

Para eliminar un archivo o varios:

```php
Storage::delete('file.jpg');
Storage::delete(['file1.jpg', 'file2.jpg']);
```

### Directorios

Para obtener un *array* con todos los archivos de una carpeta concreta:

```php
$files = Storage::files('carpeta');
```

Si queremos que además se incluyan todas las subcarpetas de la carpeta especificada:

```php
$files = Storage::allFiles('carpeta');
```

Lo mismo, pero en lugar de archivos, directorios:

```php
$dirs = Storage::directories('carpeta');

$dirs = Storage::allDirectories('carpeta');
```

Para crear un directorio (junto con todos los directorios necesarios):

```php
Storage::makeDirectory('carpeta/subcarpeta');
```

Eliminar un directorio y todo su contenido:

```php
Storage::deleteDirectory('carpeta/subcarpeta');
```

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

Este *helper* retornará la *URL* de un *asset*. La *URL* tiene como base, por defecto, el directorio donde se halla ***index.php***, es decir, el directorio ***public***. Por lo tanto, todos los *assets* que necesitemos (imágenes, *scripts .js*, etc.), deberían estar en ese directorio o en subcarpetas del mismo. Si por ejemplo tenemos imágenes en ***public/images***, la *URL* de una de esas imágenes podría ser `asset('images/logo.png')`.

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

#### action()

Genera la *URL* asociada a una acción concreta de un controlador:

```php
$url = action([MiController::class, 'send']);
```

### Varios

#### abort(), abort_if(), abort_unless()

`abort()` lanza una excepción *HTTP*, que será renderizada adecuadamente. Como primer argumento recibe el código de estado *HTTP*. Como argumentos opcionales, está el mensaje de error (*string*) y un *array* con cabeceras.

En cuanto a `abort_if()` debemos añadir como primer argumento una expresión booleana. Solo se lanzará la excepción si la condición es verdadera, justo al contrario que `abort_unless()`.

#### app()

Esta función retorna una instancia del *service container*.

#### auth()

Retorna una instancia del objeto *autenticator* sin usar la *facade* ***Auth***.

```php
user = auth()->user();
```

Se le puede pasar, como primer argumento, el nombre del *guard* deseado.

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

Los datos antiguos están disponibles a través del *helper* `old()`.

#### bcrypt()

Retorna un *hash* (no desencriptable) del *string* de entrada. Alternativa a la *facade* ***Hash***.

#### collect()

Crea una colección a partir del valor recibido.

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

#### dd()

Esta función acepta una o más expresiones como argumentos, que presenta en pantalla, y termina la ejecución inmediatamente.

#### dump()

Al igual que `dd()`, acepta una o más expresiones como argumentos, que presenta en pantalla, y pero en este caso la ejecución prosigue normalmente.

#### env()

Retorna el valor de una variable del sistema en ***\$_ENV***, ***\$_SERVER***, o en archivo ***.env*** (este último tiene prioridad). Acepta un segundo argumento con valor por defecto.

```php
$valor = env('REMOTE_USER', 'Pepito');
```

#### event()

Despacha el evento indicado.

#### old()

Proporciona acceso a doatos *flashed* en la anterior *request*.

```php
$antiguo_nombre = old('nombre');
```

Podemos pasarle un segundo argumento con un valor por defecto en caso de que no encuentre el dato. De este modo, evitaremos tener que usar el operador `??` en la plantilla *Blade*.

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

#### resolve()

Sirve para extraer un servicio de *service container*:

```php
$objeto = resolve(UnaClaseOInstancia::class)
// Equivale a:
$objeto = app()->make(UnaClaseOInstancia::class);
```

#### response()

El *helper* `response()` retorna una instancia de una respuesta del servidor.

#### session()

Se usa para obtener o establecer valores de la sesión. Estos valores se mantienen en las siguientes *requests* (esa es la idea de la sesión).

Un solo argumento *string* indica una clave de la que deseamos obtener el valor. En ese caso admite un segundo argumento con un valor por defecto.

Si se le pasa un *array*, podemos **establecer** uno o más valores.

Sin argumentos retorna el almacén completo de la sesión.

```php
$value = session('key');  // obtener valor
session(['chairs' => 7, 'instruments' => 3]);  // establecer valores
// Equivalente:
$value = session()->get('key');
session()->put(['chairs' => 7, 'instruments' => 3]);
```

#### view()

Retorna una instancia de la vista cuyo nombre le pasamos como argumento.

## Cliente *HTTP*

*Laravel* proporciona una sencilla *API* alrededor del cliente *HTTP Guzzle*, que se incluye por defecto. En caso de haberlo eliminado de las dependencias, hay que volverlo a requerir:

```
composer require guzzlehttp/guzzle
```

El punto de acceso principal es la *facade* ***Illuminate\\Support\\Facades\\Http***. La acción más simple es realizar una simple *request GET*:

```php
$resp = Http::get('http://dominio.com');
```

Esto retorna un objeto del tipo respuesta *HTTP* (***Illuminate\\Http\\Client\\Response***). Para acceder al contenido de tal respuesta, contiene varios métodos útiles:

```php
$resp->body() : string;
$resp->json($key = null) : array|mixed;
$resp->object() : object;
$resp->collect($key = null) : Illuminate\Support\Collection;
$resp->status() : int;
$resp->ok() : bool;
$resp->redirect(): bool;
$resp->header($header) : string;
$resp->headers() : array;
```

Para obtener uno de los valores de una respuesta *JSON*, estas dos sentencias son equivalentes:

```php
$nombre = $resp->json('nombre');
$nombre = $resp['nombre'];
```

Si queremos pasar argumentos a una petición *GET*, tenemos dos opciones: añadir una *query string* a la *URL* que pasamos al método `get()`, o pasar como segundo argumento un *array* asociativo con los argumentos y sus valores.

Por otro lado, existen los métodos `post()`, `put()`, `patch()` y `delete()`, que funcionan igual que `get()`, aunque para pasarles argumentos solo puede hacerse a través de un segundo argumento con el *array* asociativo correspondiente. En cambio, `head()` también acepta *query string*.

### Archivos

Para enviar un archivo, hay que enviar una solicitud *multi-part* con método *POST*:

```php
$resp = Http::attach(
    'anexo', file_get_contents('foto.jpg'), 'foto.jpg'
)->post('http://ejemplo.com/anexos');
```

El primer argumento al método `attach()` es el nombre del campo archivo del formulario que estamos enviando. El segundo es el contenido del archivo. Como tercer argumento, opcional, el nombre del archivo.

### Encabezados

Para incluir *headers* en la petición, usaremos el método `withHeaders()`, al que pasaremos un *array* con los encabezados y sus valores:

```php
$resp = Http::withHeaders([
    'X-Primero' => 'contenido del primer header',
    'X-Segundo' => 'contenido del segundo'
])->post('http://ejemplo.com/users', ['name' => 'Taylor']);
```

### Tipo de respuesta

Para indicar que la respuesta debe ser de tipo *JSON*:

```php
$resp = Http::acceptJson()->get('http://ejemplo.com/usuarios');
```

### Autenticación

Para indicar las credenciales *basic* o *digest*:

```php
// Autenticación basic:
$resp = Http::withBasicAuth('nombreusuario', 'password')->post(/*...*/);

// Autenticación digest:
$resp = Http::withDigestAuth('nombreusuario', 'password')->post(/*...*/);
```

### Timeout y reintentos

Se puede especificar un *timeout*, en segundos:

```php
$resp = Http::timeout(3)->get(/*...*/);
```

Para indicar el número de reintentos:

```php
$resp = Http::retry(3, 100)->get(/*...*/);
```

El primer argumento a `retry()` es el número máximo de reintentos, mientras que el segundo es el tiempo, en milisegundos, que se esperará entre intentos.

### Errores

El objeto respuesta posee varios métodos para el tratamiento de errores:

`successful()` retorna ***true*** si el código de respuesta es un 2xx. `failed()` hace lo propio para códigos 4xx y 5xx. `clientError()` es verdadero si la respuesta es 4xx, y `serverError()` si es 5xx.

El método `onError()`, al que debe pasarse un *callable* como argumento, ejecutará dicho callable si existe algún tipo de error (de cliente o servidor).

## Desarrollo de paquetes

Cuando desarrollamos un paquete, lo normal es incluir **todo** (rutas, vistas, *middlewares*...) dentro de una sola carpeta, por ejemplo ***src***. Lo normal es que en la raíz de esa carpeta ***src*** incluyamos un *service provider* que registre todas esas rutas, vistas, etc. Resulta conveniente que todo lo que registre ese *service provider* defina rutas de archivo relativas a la ubicación de tal *provider* (mediante el uso de ***\_\_DIR\_\_***).

Al incluir nuestro paquete dentro de un proyecto, este quedará copiado en ***vendor/<creador>/<paquete>***. Normalmente consistirá en la carpeta ***src***, junto a otros archivos, como la licencia, o un ***composer.json*** con las dependencias del propio paquete y otras configuraciones.

Cuando el paquete se incluya en otro proyecto (mediante `composer`, típicamente), se copiarán los archivos específicos en ***vendor***, y típicamente se deberá añadir (a mano) una referencia al *service provider* del paquete en la lista de *service providers* a cargar (en ***config/app.config***). Sin embargo, si no queremos que el usuario haga esta operación a mano, podemos definir el *service provider* (o más de uno) en el ***composer.json*** **del paquete**, en la sección ***extra*** (que también se puede usar para definir alias de nuestras *facades*):

```json
"extra": {
    "laravel": {
        "providers": [
            "Barryvdh\\Debugbar\\ServiceProvider"
        ],
        "aliases": {
            "Debugbar": "Barryvdh\\Debugbar\\Facade"
        }
    }
},
```

Al configurarlo así, cuando el paquete se instale se aplicará el **autodescubrimiento** del paquete, y no habrá que configurarlo a mano.

Sin embargo, a veces nos interesará que no se aplique tal autodescubrimiento en algún paquete concreto. Si es el caso, en el ***composer.json*** **del proyecto global** (no el del paquete) habrá que indicarlo, con una lista de paquetes que excluir de tal autodescubrimiento (también en la sección ***extra***):

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "barryvdh/laravel-debugbar"
        ]
    }
},
```

Si indicamos el nombre `"*"` se excluyen **todos** los paquetes del autodescubrimiento.

### Namespace

Por otro lado, hay que definir el *namespace* (o *namespaces*) del paquete en la sección de *autoload* del ***composer.json***.

```json
"autoload": {
    "psr-4": {
        "Barryvdh\\Debugbar\\": "src/"
    }
}
```

### Publicación

Tras ser incluido y registrado (o autodescubierto) el *service provider*, en ocasiones son necesarios algunos archivos extra, fuera del árbol de directorios del paquete (por ejemplo, un archivo de configuración en la carpeta ***config*** del proyecto). En este caso, hay que aplicar otro paso: la publicación del paquete mediante `artisan`:

```
php artisan vendor:publish --provider=<NombreServiceProvider>
```

El *service provider* se refiere al (o a los) *provider(s)* de nuestro paquete, que deberá(n) invocar al método `publishes()` dentro de la definición del método `boot()`. Hay que indicar el *provider fully qualified* en la línea de comandos:

```
php artisan vendor:publish --provider=Barryvdh\\Debugbar\\ServiceProvider
```

```php
$this -> publishes([__DIR__ . '/path/to/folder' => public_path('/folder/destino'),
    __DIR__ . 'path/to/file' => config_path('nombreArchivo')]);
```

En el ejemplo, se copiará la carpeta ***/path/to/folder***, que se localizará en ***public/folder/destino*** dentro del proyecto.

El método `publishes` toma un segundo parámetro opcional, que es un *tag* para categorizar el tipo de publicación. Algunos *tags* habituales son ***public***, ***config***, ***migrations***, etc. Pero podemos definir nuestros *tags* específicos.

```
php artisan vendor:publish --tag=config
```

En este ejemplo se publicarán todos los archivos y directorios con la etiqueta ***public*** de todos los proveedores que la definan.

Si no especificamos *provider* ni *tag*, nos aparecerá un menú con las opciones a publicar.

Para forzar una publicación aunque ya exista, se añadirá `--force` para sobrescribir.

Para publicar parte de la configuración dentro del archivo de configuración del paquete se usa `mergeConfigFrom()`. Supongamos que nuestro archivo de configuración es ***courier.php***.

```php
$this->mergeConfigFrom(__DIR__.'/path/to/config/courier.php', 'courier');
```

Este ejemplo fusiona todas las claves del archivo ***courier.php*** en el archivo correspondiente publicado ya. Solo añadirá las claves que no estén publicadas. Solo actúa en el primer nivel, es decir, si el valor de una clave es un *array*, no entrará a mirar qué claves tiene.

Este otro ejemplo fusiona solo el contenido de una clave (***key***):

```php
$this->mergeConfigFrom(__DIR__.'/path/to/config/courier.php', 'courier.key');
```

### Carga de recursos

Si nuestro paquete tiene rutas, se pueden cargar así, dentro de la definición de `boot()` de nuestro *service provider*:

```php
$this->loadRoutesFrom($rutaArchivo);
```

Es importante recordar que las rutas en el directorio estándar (***routes***) de la aplicación, se cargan automáticamente a través del ***RouteServiceProvider*** por defecto de *laravel*. Este proveedor, les aplica a estas, el *middleware* ***web*** o ***api***, según sea el caso. Pero este *middleware* no se aplica a rutas cargadas por nuestro proveedor, con lo que deberíamos aplicarlo a mano:

```php
Route::middleware('web')
    -> group(function() {
        // Carga de rutas:
        $this->loadRoutesFrom(__DIR__ . '/routes/web.php');
        // Equivaldría a definir aquí las rutas con Route::...
    });
```

Por simplicidad, esto es equivalente a:

```php
Route::middleware('web')
    -> group(__DIR__ . '/routes/web.php');
```

En el siguiente ejemplo definimos las rutas *API* para controladores en el *namespace* ***Barryvdh\Debugbar\Controllers***:

```php
Route::namespace('Barryvdh\Debugbar\Controllers')  // se buscarán en este namespace todos los controladores involucrados
    -> middleware('api')
    -> prefix('/api')
    -> group(__DIR__ . '/routes/api.php');
```

Para cargar traducciones y vistas en directorios no estándar:

```php
$this -> loadTranslationsFrom($rutaDirectorio, $proyecto);
$this -> loadViewsFrom($rutaDirectorio, $proyecto);
```

La variable ***\$proyecto*** es un *string* con el nombre del proyecto. Si su valor es, por ejemplo ***Debugbar***, entonces para acceder, por ejemplo a la vista ***home***, se haría así:

```php
view('Debugbar::home');
```

Para registrar migraciones:

```php
$this -> loadMigrationsFrom($rutaDirectorio);
```
