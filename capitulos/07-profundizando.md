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

## *File storage*

Si queremos permitir la subida de archivos a la aplicación, debemos, primero, tener en cuenta que el formulario de subida debe ser de tipo ***POST***, y que deberá contener el atributo *HTML* `enctype="multipart/form-data"`.

Por otro lado, *PHP* debe estar configurado para permitir las subidas de archivos, con lo que el archivo ***php.ini*** deberá contener la línea `file_uploads = On`.

Finalmente, incluiremos nuestros campos *HTML* `<input type="file" ...>`.

En *Laravel*, la configuración de archivos se encuentra en ***config/filesystems.php***, en el que podemos definir nuestros "discos", caracterizados por un *driver* (admitidos 'local', 'ftp', 'sftp' y 's3') y una ubicación concreta.

El *driver* 'local' es el único que no precisa de la instalación de paquetes extra, y nos sirve para almacenar los archivos en las carpetas locales del proyecto. Por convenio y claridad, se suele usar el directorio ***storage*** y sus subcarpetas.

El archivo también define el disco por defecto.

Si queremos definir un disco público, de tal modo que los archivos allí subidos sean accesibles públicamente (a través de su *URL*), usaremos el disco ya preconfigurado por defecto llamado ***public*** (podemos editarlo o cambiarle el nombre). Utiliza el *driver* local, y por defecto está asociado a ***storage/app/public***. Para que se pueda acceder a través de *URL* a sus archivos hay que crear un enlace simbólico en ***public***. Esto se puede hacer fácilmente con el comando:

```
php artisan storage:link
```

Así, se creará el enlace simbólico ***public/storage***, que enlaza con ***storage/app/public***. A partir de entonces ya podemos acceder a los archivos mediante cosas del tipo `asset('storage/archivo.txt')`.

Este enlace queda definido en el archivo de configuración:

```php
'links' => [
    public_path('storage') => storage_path('app/public');
]
```

Así, podemos definir otros enlaces simbólicos, bien editando ***filesystems.php*** o bien usando `artisan`, que cambiará ese archivo de configuración adecuadamente.

### Operaciones con archivos

Las operaciones de disco se realizan mediante la *facade* ***Storage***. Todos los *paths* son relativos a la raíz de nuestro disco definido en la configuración.

Suponiendo que la variable ***$filecontents*** almacene el contenido de un archivo, para guardarlo en la carpeta ***avatars*** de nuestro disco llamado ***local***:

```php
Storage::disk('local') -> put('avatars', $filecontents);
```

Si simplemente queremos guardarlo en nuestro disco por defecto:

```php
Storage::put('avatars', $filecontents);
```

Para obtener el contenido *raw* de un archivo:

```php
Storage::get('textos/texto5.txt');
```

Para saber si un archivo existe o no existe:

```php
if(Storage::disk('disco1') -> exists('texto.txt'))
    /* ... */
if(Storage::disk('disco5') -> missing('texto.txt'))
    /* ... */
```

### Descarga y otros

Podemos generar una *response* que obligue al cliente a descargar un archivo concreto:

```php
return Storage::download('catalogo.pdf');
```

El método `download()` admite un segundo argumento, opcional, con el nombre con el que aparecerá el archivo al cliente. Un tercer argumento, también opcional, será un *array* con cabeceras *HTTP*.

Para obtener la *URL* de un archivo se usa el método `url()` de ***Storage***, que en *driver* local retorna una *URL* relativa al archivo. En driver 's3' (*cloud storage*) retorna una *URL* absoluta.

Existen otros métodos para obtener información del archivo en disco, como `size()` o `lastModified()`.

### Almacenamiento

Se pueden almacenar archivos, como hemos visto, con el método `put()`, especificando el directorio, y el contenido. Dicho contenido puede ser el contenido en bruto, o un recurso *PHP*.

Para acceder a uno de los archivos subidos en la *request* actual:

```php
$archivo = $request -> file('docu');
```
En este caso, el archivo se ha subido en un formulario donde el *input* ***docu*** se correspondía a un archivo. Para almacenar ese archivo:

```php
$ruta = $request -> file('docu') -> store('archivos');
```

Lo cual guardará ese archivo en la carpeta ***archivos*** del disco por defecto. Sin embargo, esto se guardará con un nombre único en disco, y una extensión basada en el tipo *MIME* del archivo. El método retornará el *path* completo con el nombre que recibirá el archivo en disco.

La sentencia anterior es equivalente a:

```php
$ruta = Storage::putFile('archivos', $request -> file('docu'));
```

Si queremos que guarde el archivo con un nombre concreto:

```php
$ruta = $request -> file('docu') -> storeAs('archivos', 'archivo55.txt');

$ruta = Storage::putFileAs('archivos', $request -> file('docu'), 'archivo55.txt');
```

Las dos sentencias son equivalentes.

Si queremos especificar un disco concreto, en el caso de usar ***Storage*** ya sabemos cómo se hace. En el caso de usar `store()`, el nombre del disco se incluirá en un segundo argumento del método. En el caso de `storeAs()`, en un tercer argumento.

### Información de archivos

En el caso de archivos subidos en la *request*, podemos obtener otra información de los mismos:

```php
$tipo = $request -> file('docu') -> getMimeType();  // tipo MIME
$size = $request -> file('docu') -> getSize();  // tamaño (bytes)
$tipo = $request -> file('docu') -> getClientOriginalName();  // nombre original
$tipo = $request -> file('docu') -> extension();  // extensión
```

Para saber si en la *request* actual viene un archivo:

```php
if($request -> hasFile('docu'))
    /* ... */
```

### Visibilidad

Para almacenar archivos para que sean accesibles públicamente (desde otras aplicaciones, navegadores, etc.), hay que marcarlos como públicos. Por ejemplo, pasando un tercer argumento 'public' a `put()`, o usando métodos como `storePublicly()` o `storePubliclyAs()`.

Para cambiar la visibilidad de archivos ya almacenados:

```php
$visibility = Storage::getVisibility('file.jpg');
Storage::setVisibility('file.jpg', $vis);
```

Aquí, ***$vis*** puede ser ***'public'*** o ***'private'***.

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

Aunque lo mejor es usar el segundo argumento (valor por defecto) de `old()`:

```html
<input type="text" id="ciudad" name="ciudad" value="{{ old('ciudad', 'Beijing') }}">
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

#### session()

Se usa para obtener o establecer valores de la sesión. Estos valores se mantienen en las siguientes *requests* (esa es la idea de la sesión).

Un solo argumento *string* indica una clave de la que deseamos obtener el valor. En ese caso admite un segundo argumento con un valor por defecto.

Si se le pasa un *array*, podemos **establecer** uno o más valores.

Sin argumentos retorna el almacén completo de la sesión.

```php
$value = session('key');  // obtener valor
session(['chairs' => 7, 'instruments' => 3]);  // establecer valores

$value = session() -> get('key');
session() -> put(['chairs' => 7, 'instruments' => 3]);
```

Si queremos que un valor concreto esté disponible **únicamente** en la *request* actual y en la siguiente, debemos usar el método `flash()`:

```php
session() -> flash('estado', 'archivo enviado con éxito');
```

Si en la siguiente queremos darle una *request* extra de duración, usaremos el método `reflash()`.

## Desarrollo de paquetes

### Publicación de la configuración

Una vez hayamos incluido un paquete en nuestro proyecto con `composer`, necesitamos publicar la configuración del mismo, dentro del árbol de directorios de nuestro proyecto (es decir, fuera del subdirectorio de ***vendor*** donde resida el paquete). Los archivos de configuración que serán publicados están definidos en el *service provider* que viene con el paquete.

Para publicar la configuración de todos los paquetes que no lo hayan hecho:

```
php artisan vendor:publish
```

Para publicar la configuración de un paquete concreto usaremos la opción `--provider`.
