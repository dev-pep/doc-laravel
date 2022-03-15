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

## Localización

Soporte multilenguage incorporado en *Laravel*.

Los *strings* de idiomas están en el directorio ***lang***. En este directorio deben ubicarse las carpetas de cada lenguage: ***en***, ***es***, etc. En estas carpetas se almacenan los archivos de *strings*.

Lo único que hacen los archivos de *strings* es retornar un *array* cuyos elementos tienen como clave un *string* que sirve como identificador del *string* con el texto concreto.

El lenguaje (*locale*) por defecto está definido en ***config/app.php***, pero se puede establecer en *run time* mediante la *facade* ***App***:

```php
Route::get('welcome/{locale}', function($locale) {
    if(in_array($locale, ['en', 'es', 'fr']))
        App::setLocale($locale);
});
```

En este caso, el *locale* queda definido **para la** ***request*** **actual únicamente**.

El código identificador de un *locale* concreto debe seguir la nomenclatura de *ISO 15897*, por lo que si es un lenguaje con diferencias territoriales, el formato es del tipo ***en_GB*** (no ***en-gb***).

También hay un *fallback locale*, definido en ***config/app.php*** (clave ***fallback_locale***), cuyos *strings* se usan cuando no existe traducción de un *string* en el *locale* actual.

Para saber cuál es el *locale* actual, se usa `App::currentLocale()`. Para saber si el *locale* actual se corresponde con uno específico, se puede usar:

```php
if(App::isLocale('en'))
    //...
```

### Definición de los *strings*

Se deben crear los archivos deseados dentro de los directorios de cada *locale*. Cada archivo debe retornar un *array* con la clave (*ID* o *short key* de cada *string*).

Por ejemplo, en ***lang/es/mensajes.php*** podríamos tener:

```
<?php
    return [
        'bienvenida' => 'Bienvenido a nuestra aplicación.',
        'gracias' => 'Gracias por visitar nuestra web.'
    ];
```

En el fichero de traducción al inglés (***lang/en/mensajes.php***), tendríamos:

```
<?php
    return [
        'bienvenida' => 'Welcome to our application.',
        'gracias' => 'Thanks for visiting our website.'
    ];
```

Y el acceso a los *strings* se haría de la siguiente forma:

```php
echo __('mensajes.bienvenida');
```

En este caso, la función `__()` retornará el mensaje de bienvenida en el idioma del *locale* actual. Si no existe traducción para dicho *locale*, la función simplemente retornará la clave (en este caso, ***mensajes.bienvenida***).

Dentro de una plantilla *Blade*, pueden incluirse *strings* de dos formas (equivalentes):

```
{{ __('mensajes.bienvenida') }}
```

Cuando las aplicaciones se vuelven grandes, guardar las traducciones mediante *short keys* se puede llegar a hacer muy farragoso y confuso. Es por ello que también se pueden definir las traducciones utilizando como clave la "traducción por defecto". Estos archivos se almacenan como *JSON* en el directorio ***lang*** directamente, y tendrán como nombre el nombre del *locale* (y extensión *json*). Por ejemplo, el archivo ***lang/en.json*** podría contener:

```json
{
    "Bienvenido a nuestra aplicación.": "Welcome to our application",
    "Gracias por visitar nuestra web.": "Thanks for visiting our website."
}
```

En este caso, el acceso al *string* se hace igual, indicando la traducción por defecto (en este caso, en español):

```php
echo __('Bienvenido a nuestra aplicación.');
```

### Parámetros en los *strings*

Pueden definirse parámetros en los *strings*, prefijando dos puntos (***:***):

```php
'bienvenida' => 'Bienvenido a nuestra aplicación, :nombre.',
```

A la hora de acceder al *string* se debe añadir un *array* con los argumentos:

```php
echo __('bienvenida', ['nombre' => 'Doyle']);
```

Si en la definición del *string* el *placeholder* del parámetro está todo en mayúsculas o tiene únicamente la primera letra en mayúsculas, el valor que se le pase será cambiado para coincidir con esa capitalización.

Para traducir (o cambiar el texto) *strings* de paquetes que tengan sus propios *language files*, en lugar de editar esos ficheros del paquete, se pueden *override* creando *language files* en ***lang/vendor/\<paquete>/\<locale>/\<archivo>***.

Por ejemplo, supongamos que el paquete ***skyrim/hearthfire*** (recordemos que esto es *vendor/paquete*) tiene un archivo de lenguaje en inglés llamado ***messages.php***. Entonces solo tenemos que crear un archivo ***lang/vendor/hearthfire/es/messages.php*** con nuestros propios *strings*.

## Correo electrónico

*Laravel* proporciona una *API* para el envío de correos electrónicos, que por debajo utiliza la biblioteca *Symphony Mailer*. La configuración se halla en ***config/mail.php***. Es posible escoger entre varios *drivers* (*mailers*) para enviar correo electrónico. Cada uno de ellos tiene su propia configuración.

En el *array* ***mailers*** del archivo de configuración se encuentran todos los *mailers* disponibles, junto a su configuración. A parte, se puede definir el *mailer* por defecto, y el remitente *from* (dirección y nombre).

Nos centraremos en el envío por *SMTP*. Para otros *mailers*, véase la documentación oficial.

### Los *mailables*

Cada tipo de correo que enviamos se corresponde con una clase *mailable*. Estas clases se almacenan en ***app/Mail***. Se pueden crear mediante *artisan*:

```
php artisan make:mail PwdCaducado
```

Esta clase posee, a parte de su constructor, un método `build()`, que es donde se configura el correo a enviar. Este correo debe retornar una instancia del *mailable*. El objeto retornado contiene toda la información del correo a enviar: remitente, destinatiario(s), contenido, asunto, etc.

Para ello, lo habitual es utilizar como base la propia instancia, a través de ***\$this***, añadiéndole información a través del encadenado de métodos (no importa el orden).

#### Remitente

El método `from()` permite especificar un remitente válido. El primer argumento que recibe es in *string* con la dirección de correo del remitente, y el segundo, el nombre del mismo (opcional). Si la dirección de respuesta debe ser distinta, se puede usar el método `replyTo()`, que tiene los mismos argumentos que `from()`.

Se puede utilizar un remitente configurado globalmente en ***config/mail.php***. Además, se puede indicar la dirección *reply to* (si es distinta a *from*):

```php
'from' => ['address' => 'pepe@ejemplo.com', 'name' => 'Nombre del remitente'],
'reply_to' => ['address' => 'paco@ejemplo.com', 'name' => 'Nombre a quien respondemos'],
```

Si existe tal configuración, los métodos `from()` y `replyTo()` tienen prioridad sobre ella.

#### Asunto

Para especificar el asunto, simplemente se incluirá en la cadena el método `subject()`, que recibe un *string* con el asunto.

Si no se especifica, el asunto, por defecto, es el nombre de la clase *mailable*.

#### Contenido

Para especificar el contenido, se puede usar una plantilla *Blade*, cuyo nombre se pasará como argumento al método `view()`. Un segundo argumento podrá contener un *array* con argumentos para la plantilla.

Es posible definir una versión en texto plano mediante el método `text()`, que usa como base también una plantilla *Blade*, que se usará como texto. Recibe los mismos argumentos que `view()`.

La plantilla *Blade* utilizada tendrá acceso a todas las propiedades públicas del *mailable*, a parte de los argumentos que reciba. También se puede usar el método `with()`, al que se le pasará un *array* de argumentos para la plantilla (o plantillas: *Blade* y texto).

#### Adjuntos

Se puede añadir un adjunto mediante el método `attach()`. El primer argumento a dicho método es la ruta (completa) del archivo. Como segundo argumento, opcional, se puede pasar un *array* con el nombre que tendrá el archivo y/o el tipo del mismo.

```php
public function build()
{
    return $this->view('emails.orders.shipped')
                ->attach('/path/to/file', [
                    'as' => 'name.pdf',
                    'mime' => 'application/pdf',
                ]);
}
```

Por otro lado, el método `attachFromStorage()` toma como primer argumento la ruta de archivo **relativa** al directorio ***storage***. En este caso, el segundo argumento (opcional) será el nombre que tendrá el archivo en el correo. Como tercer argumento, también opcional, podemos incluir el *array* de opciones (con el tipo *MIME*, por ejemplo). Similar a este método, `attachFromStorageDisk()` adjunta un archivo en uno de los discos definidos. El nombre del disco es el primer argumento. El segundo argumento es la ruta absoluta del archivo dentro de ese disco. También acepta opcionalmente el nombre con el que se enviará y el *array* de opciones.

El método `attachData()` recibe como primer argumento una cadena de *bytes*, como segundo el nombre del archivo (solo nombre, no ruta). Opcionalmente puede incluirse un *array* de opciones.

### Envío de correo

Para enviar un correo, se utiliza la *facade* ***Illuminate/Support/Facades/Mail***. En este caso, debemos especificar el destinatario o destinatarios, copias, copias ocultas, etc. El método `to()` acepta un *string* con la dirección de correo electrónico, o un objeto. En el caso de un objeto, se usarán sus propiedades ***mail*** y ***name*** para el envío. Pero también acepta un *array* con *strings* y/o objetos, si queremos enviarlo a varios destinatarios.

También pueden encadenarse métodos como `cc()` (*carbon copy*, copia), `cco()` (*blind carbon copy*, copia oculta), con el mismo formato de argumentos que `to()`.

**Al final de la cadena** de métodos, se debe escribir el método `send()`, al que se le pasará una instancia del *mailable* a enviar.

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->send(new OrderShipped(33));
```

Si en lugar de usar el *mailer* por defecto deseamos usar otro:

```php
Mail::mailer('postmark')
    ->to($request->user())
    ->send(new OrderShipped($order));
```

### Previsualización de *mailables*

Para ver una previsualización de un *mailable*, no es necesario enviarlo. Simplemente si como *response* retornamos la instancia concreta de dicho *mailable*, podremos verlo en pantalla.

## Desarrollo de paquetes

Los paquetes pueden ser *standalone*, es decir, funcionan en cualquier proyecto *PHP* (incluyendo proyectos *Laravel*). Otros están específicamente diseñados para integrarse en *Laravel* (con sus rutas, vistas, controladores y configuración). Para los primeros, simplemente hay que requerirlos mediante *composer*, y solo se debe tener cuidado en que el paquete tenga una correcta configuración del apartado ***autoload*** en el archivo ***composer.json*** **del mismo paquete**, para que se produzca la autocarga de clases de forma adecuada.

En el archivo ***composer.json*** del paquete, estarán especificadas también las dependencias de otros posibles paquetes.

En este apartado veremos los pasos adicionales que se deben realizar en el desarrollo de un paquete específico para *Laravel*.

### Descubrimiento de paquetes

En el archivo ***composer.json*** **del paquete** se puede incluir la configuración destinada a *Laravel* incluyendo una subsección ***laravel*** dentro la sección ***extra***. Allí podemos indicar los proveedores de servicio que se deberían cargar en la aplicación automáticamente. De este modo, evitamos tener que añadir el nombre del proveedor en la sección ***providers*** del archivo ***config/app.php*** (aunque podríamos elegir hacerlo así, por cualquier motivo). También se pueden indicar *alias* para las *facades* que pueda definir el paquete (sección ***aliases***).

Supongamos que estamos creando un paquete ***developer/acme***. Dentro de una aplicación, este paquete se instalará en ***vendor/developer/acme***.

```json
"autoload": {
    "psr-4": {"Acme\\": "src/"},
    "psr-4": {"Acme\\Fuu\\": "src2/"}
},

"extra": {
    "laravel": {
        "providers": [
            "Acme\\ServiceProvider",
            "Acme\\Fuu\\ServiceProvider"
        ],
        "aliases": {
            "UnaFacade": "Acme\\Fuu\\UnaFacade"
        }
    }
},
```

Con este ***composer.json*** (del paquete), las clases de tipo ***Acme\\NombreClase*** se buscarán en al archivo ***vendor/developer/acme/src/NombreClase.php***. Lógicamente, este archivo contendrá:

```php
namespace Acme;
```

Por otro lado, las clases de tipo ***Acme\\Fuu\\NombreClase*** se buscarán en al archivo ***vendor/developer/acme/src2/NombreClase.php***. En el archivo:

```php
namespace Acme\Fuu;
```

Clases definidas en subdirectorios deberán seguir un esquema adecuado de *namespaces* anidados.

En el ejemplo, el paquete define dos *namespaces* y dos *service providers*. En este caso, se cargarán dichos *service providers*, en ***vendor/developer/acme/src/ServiceProvider.php*** y ***vendor/developer/acme/src2/ServiceProvider.php***.

Por otro lado, si la aplicación que requiere ese paquete no desea que este sea descubierto, y por lo tanto que no se cargue su (o sus) *service providers*, en el ***composer.json*** **de la aplicación** se incluirá:

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "developer/acme"
        ]
    }
}
```

Si la aplicación no desea que **ningún paquete** sea descubierto:

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "*"
        ]
    }
}
```

### Service providers

Cargar el *service provider* de un paquete es sinónimo de cargar el paquete. Pero, ¿qué es lo que hacen estos *services providers*?

- Registrar en el *service container* los servicios (clases) que proporciona el paquete (método `register()`).
- Informar a *Laravel* acerca de dónde encontrar los recursos que proporciona (vistas, localización, configuración, etc.), mediante el método `boot()`.

### Recursos

#### Configuración

Una vez requerido e instalado el paquete, es frecuente que su configuración deba publicarse (en el directorio ***config***). Para ello habrá que invocar el método `publishes()` del *service provider*, pasándole la configuración concreta.

```php
public function boot()
{
    $this->publishes([
        __DIR__.'/../config/courier.php' => config_path('courier.php'),
    ]);
}
```

Para publicar la configuración tras requerir el paquete, usaremos *artisan*:

```
php artisan vendor:publish --provider=<NombreServiceProvider>
```

El nombre del *service provider* debe indicarse *fully qualified*.

En el caso del método `boot()` del ejemplo, el contenido del archivo especificado se copiará en la carpeta de configuración de la aplicación, con el nombre indicado (la ruta especificada es relativa a ***\_\_DIR\_\_***, es decir, relativa al *service provider*).

El método `publishes` toma un segundo parámetro opcional, que es un *tag* para categorizar el tipo de publicación. Algunos *tags* habituales son ***public***, ***config***, ***migrations***, etc. Pero podemos definir nuestros *tags* específicos.

```
php artisan vendor:publish --tag=config
```

Si se omite el *flag* `--provider` y el `--tag`, aparecerá un menú para elegir el proveedor o *tag* deseado.

La publicación no sobrescribe archivos ya existentes. Para forzar una publicación aunque ya exista el archivo, se añadirá `--force` para sobrescribir.

Evidentemente, lo habitual es que el destino de las configuraciones sea el directorio ***config***, pero de hecho se puede indicar cualquier otro directorio de la aplicación.

Por otro lado, en lugar de publicar el archivo como un bloque, se pueden publicar únicamente las opciones (configuraciones) que no existan en la configuración de la aplicación. De este modo, el usuario del paquete puede decidir qué configuraciones serán añadidas y cuáles no, configurando a mano, antes de publicar, las opciones que desea mantener. Para ello, se usará el método `mergeConfigFrom()`, que recibe, por una parte, el archivo de configuración de nuestro paquete, y por otra, el **nombre** del archivo de configuración (sin ruta ni extensión):

```php
public function register() {
    $this->mergeConfigFrom(__DIR__ . '/../config/courier.php', 'courier');
}
```

Este ejemplo fusiona todas las claves del archivo ***courier.php*** en el archivo correspondiente publicado ya. Solo añadirá las claves que no estén publicadas. Solo actúa en el primer nivel, es decir, si el valor de una clave es un *array*, no entrará a mirar qué claves tiene.

> IMPORTANTE: este método debe llamarse desde el método `register()` del *service provider*, no desde `boot()`.

#### Rutas

Para cargar un archivo con rutas, se debe usar el método `loadRoutesFrom()` del proveedor de servicios.

```php
public function boot() {
    $this->loadRoutesFrom(__DIR__ . '/../routes/rutas.php');
}
```

Esto equivale a definir las rutas mediante la *facade* `Route`. Se podrían definir las rutas dentro de `boot()` usando esa *facade*, pero queda mucho más claro haciéndolo mediante el archivo de rutas.

Es importante recordar que por defecto, las rutas en el directorio estándar (***routes***) de la aplicación, se cargan automáticamente a través del ***RouteServiceProvider*** por defecto de *Laravel*. Este proveedor aplica el *middleware* ***web*** o ***api*** a estas rutas, según sea el caso. Pero este *middleware* no se aplica a las rutas que cargamos mediante nuestro proveedor de servicios, con lo que en caso de interesar, debemos indicarlo específicamente:

```php
public function boot() {
    Route::middleware('web')
        -> group(function() {
            $this->loadRoutesFrom(__DIR__ . '/../routes/rutas.php');
    });
}
```

Por simplicidad, esto es equivalente a:

```php
public function boot() {
    Route::middleware('web')
        -> group(__DIR__ . '/../routes/rutas.php');
}
```

#### Migraciones

Para registrar migraciones:

```php
public function boot() {
    $this -> loadMigrationsFrom('/../database/migrations');
}
```

De esta forma, al ejecutar `php artisan migrate` se ejecutarán las migraciones correspondientes, sin necesidad de copiarlas al directorio ***database/migrations*** de la aplicación.

#### Traducciones

Para publicar traducciones, se usa el método `loadTranslationsFrom()` del *service provider*. Este método necesita como primer argumento el directorio del paquete donde se encuentran los archivos de traducción, y como segundo argumento el nombre del proyecto, que se utilizará como prefijo para acceder a estas traducciones, para que no interfiera con las traducciones definidas en la misma aplicación.

Para acceder a estas traducciones, se usa el formato ***paquete::archivo.texto***.

```php
public function boot()
{
    $this->loadTranslationsFrom(__DIR__ . '/../lang', 'courier');
}
```

En este caso, accederíamos a la traducción ***welcome*** del archivo ***mensajes*** de este modo:

```php
echo trans('courier::mensajes.welcome');
```

#### Vistas

Para publicar vistas se usa el método `loadViewsFrom()`:

```php
public function boot()
{
    $this->loadViewsFrom(__DIR__ . '/../resources/views', 'courier');
}
```

En este caso, hay que especificar dos parámetros: la ruta del directorio del paquete que contiene las rutas, y el nombre del paquete, que se usará como prefijo para que no haya colisiones con las vistas de la aplicación.

De este modo, accederemos a la vista ***home*** del paquete de esta forma:

```php
view('courier::home');
```

#### *View components*

Si el paquete contiene *view components*, utilizaremos el método `loadViewComponentsAs()` del *service provider*. Este método recibe dos argumentos: el nombre del paquete y un *array* con los nombres *fully qualified* de las clases correspondientes.

```php
use Courier\Components\Alert;
use Courier\Components\Button;

public function boot()
{
    $this->loadViewComponentsAs('courier', [
        Alert::class,
        Button::class,
    ]);
}
```

Una vez publicados los componentes se puede hacer uso de ellos de esta forma:

```html
<x-courier-alert />

<x-courier-button />
```

#### Otros

Para publicar cualquier otro tipo de archivo, como *assets* públicos, simplemente se puede usar el método `publishes()`. Por otro lado, si deseamos separar la publicación en grupos separados de archivos podemos utilizar etiquetas e invocar este método varias veces, cada una con una etiqueta distinta.

## Planificación de tareas

Es posible planificar tareas que se realicen de forma periódica en el servidor. El planificador se encuentra en la clase ***app/Console/Kernel.php***, en el método `schedule()`, el cual recibe una instancia de la clase ***Illuminate\Console\Scheduling\Schedule***. Esta instancia contiene los métodos necesarios para registrar las tareas a ejecutar.

El método `call()` del objeto ***Schedule*** recibe cualquier *callable PHP* como primer argumento. Si este precisa a su vez argumentos, se pueden pasar en un *array* como segundo argumento de `call()`.

El objeto retornado por este método `call()` tiene métodos que permiten definir la periodicidad con la que se debe ejecutar el *callable*. Algunos de estos métodos son `everyMinute()`, `everyFiveMinutes()`, `everyFifteenMinutes()`, `everyThirtyMinutes()`, `hourly()`, `ourlyAt()` (recibe como argumento el minuto, de 0 a 59), `everyTwoHours()`, `daily()`, `dailyAt()` (argumento *string* del tipo ***13:30***), `twiceDaily()` (dos argumentos numéricos con la hora, de 0 a 23), `weekly()` (se ejecuta el domingo a las 0:00), `weeklyOn()` (dos argumentos: día de la semana de 0 a 6 empezando en domingo, y *string* con la hora de tipo ***13:30***), `monthly()` (día 1 a las 0:00), `monthlyOn()` (argumentos: día del mes y *string* con la hora), `twiceMonthly()` (día primero, día segundo y *string* con la hora), `lastDayOfMonth()` (*string* con la hora), `quarterly()` (primer día del cuatrimestre, a las 0:00), `yearly()` (1 de enero a las 0:00), `yearlyOn()` (mes, día, *string* con la hora) o `timezone()` (*string* con el *timezone*).

```php
protected function schedule(Schedule $schedule)
{
    $schedule->call(function () {/* ... */})
        ->daily();
}
```

De forma similar a `call()` puede usarse el método `command()`, que recibe un *string* con un comando *artisan*, o `exec()`, que recibe un *string* con un comando *shell*.

Junto a los métodos de periodicidad, puede encadenarse uno o varios métodos que restringen esa periodicidad. Algunos de estos métodos son `weekdays()`, `weekends()`, `sundays()` hasta `saturdays()`, `days()` (recibe un *array* con días de la semana de 0 a 6 empezando en domingo), `between()` (recibe dos *strings* con hora inicial y final), `unlessBetween()` (2 *strings* con hora inicial y final) o `when()` (recibe una *closure* que retorna un booleano).

Para ver qué acciones están planificadas se puede usar el comando *artisan*:

```
php artisan schedule:list
```

Las tareas planificadas no se ejecutan en modo de mantenimiento. Sin embargo, si se desea que alguna de ellas se ejecute incluso en dicho modo, hay que encadenar el método `evenInMaintenanceMode()` al definirla.

### Ejecución del planificador

Las tareas se ejecutan en el servidor cuando se invoca el comando *artisan* `schedule:run`. Esto evaluará qué tareas deben ejecutarse según su periodicidad. Por lo tanto, se deberá configurar una entrada `cron` en el servidor que ejecute cada minuto este comando:

```
php artisan schedule:run
```

En el entorno local en el que desarrollamos la aplicación no es necesario agregar una entrada `cron`. Podemos simplemente ejecutar:

```
php artisan schedule:work
```

Esto simplemente ejecutará `schedule:run` cada minuto (no lo hace desde el *background*, con lo que para detenerlo simplemente se deberá presionar ***Ctrl-C***).
