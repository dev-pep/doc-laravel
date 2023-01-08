# Arquitectura

## Ciclo de vida de una petición

El punto de entrada es el archivo ***public/index.php***. Este carga la definición del *autoloader* de *Composer* y obtiene una instancia de la aplicación (el *service container*) a través de ***bootstrap/app.php***.

Luego, la petición entrante es enviada (dependiendo del tipo de *request*) a uno de los *Kernels*: el *HTTP* (***app/Http/Kernel***) o el de consola (***app/Console/Kernel***).

En el caso del *kernel HTTP*, antes de manejar la petición, define un *array* de *bootstrapers* que realizan una serie de tareas, como configurar el *logging* o el tratamiento de errores, o detectar el entorno. Esto es interno de *Laravel*, por lo que no hay que preocuparse por ello. Además, este *kernel* define también una lista de *middleware HTTP* por el que la petición tendrá que pasar antes de ser tratada por la aplicación. Este *middleware* maneja la lectura/escritura en la sesión *HTTP*, determina si la aplicación está en modo de mantenimiento, verifica el *token CSRF*, etc. Luego, pasará la petición a la aplicación a través de su método `handle()`, que simplemente recibe una *request* y retorna una *response* (básicamente todo el proceso de la aplicación).

Lo más importante que realizan estos *bootstrapers* es registrar (cargar) los *service providers* de la aplicación. Estos están listados en ***config/app.h***, en el *array* ***providers***. En este caso, son instanciados, uno por uno, y se llama al método `register()` de cada uno de ellos, lo cual registra todos los **servicios** (véase más abajo). Una vez se han ejecutado todos sus métodos `register()`, se invocan sus métodos `boot()`, que pueden realizar otras tareas.

Los *service providers* por defecto están en ***app/Providers***, y registran todos los servicios necesarios: base de datos, colas, validación, etc.

Una vez han sido cargados todos los *service providers*, se pasará la petición al *router* para que sea manejada.

## *Routing*

Uno de los *service providers* más importantes es ***App\\Providers\\RouteServiceProvider***, que carga los archivos de rutas presentes en el directorio ***routes***. Tras esto, cada petición entrante será despachada por el enrutador hacia una ruta o controlador, pasando por el *middleware* necesario.

Una vez procesada la *request*, el controlador o ruta retornará una *response*, que pasará a su vez por otro *middleware* (de "salida") el método `handle()` del *kernel* retorna esa *request*. Finalmente, ***index.php*** envía esa respuesta al navegador del usuario.

## Servicios e inyección de dependencias

Para entender este mecanismo, debemos comprender lo que son los servicios, los clientes, la dependencia y la inyección de dependencias (*dependency injection*).

### Inyección de dependencias

Se denomina **servicio** a una clase que es utilizada por otra clase (llamada clase **cliente**). Entonces, se dice que la clase cliente **depende** de la clase servicio.

Supongamos que tenemos una clase ***Viaje***, que en alguna parte de su lógica utiliza una instancia de una clase ***Coche***. Esto podría verse así:

```php
class Coche
{
    public function __construct($marca, $modelo) { /*...*/ }
}

class Viaje
{
    private $medio;

    public function __construct()
    {
        $this->medio = new Coche('Tesla', 'Model X');
    }
}

$viaje = new Viaje;
```

En este caso, la clase ***Coche*** es un servicio utilizado por la clase cliente ***Viaje***. Existe en este código una fuerte dependencia de la clase ***Coche*** por parte de ***Viaje***. Esto presenta varios problemas:

- La clase cliente debe conocer los detalles de instanciación del servicio, lo cual resta mantenibilidad y reusabilidad, y además dificulta las pruebas de código (para hacer un test unitario sobre la clase ***Viaje*** debe probarse obligatoriamente la clase ***Coche***).
- Si se desea cambiar algún parámetro de ***Coche*** habrá que modificar el código de ***Viaje***.
- Si en lugar de utilizar la clase ***Coche*** deseamos utilizar otra clase ***Bicicleta***, también habrá que modificar la clase cliente.

La inyección de dependencias consiste, pues, en liberar al cliente de los detalles de instanciación de los servicios, de tal modo que tal instanciación se produzca **fuera** de la clase:

```php
class Coche
{
    public function __construct($marca, $modelo) { /*...*/ }
}

class Viaje
{
    private $medio;

    public function __construct(Coche $coche)
    {
        $this->medio = $coche;
    }
}

$coche = new Coche('Tesla', 'Model X');
$viaje = new Viaje($coche);
```

En el constructor de ***Viaje*** indicamos el tipo del parámetro (*type hint*), lo cual no es necesario, pero es una buena práctica acostumbrarse a hacerlo cuando se inyecten dependencias.

Ahora los detalles del servicio quedan completamente fuera del código de ***Viaje***.

La inyección de dependencias aumenta la flexibilidad del código gracias a la posibilidad de inyectar interfaces. Por ejemplo, supongamos que la clase cliente utiliza los métodos públicos `getData()` y `setData()` del servicio. Entonces, cualquier clase que implemente obligatoriamente esos dos métodos, podría inyectarse en lugar de únicamente ***Coche***:

```php
interface Vehiculo
{
    public function getData();
    public function setData($data);
}

class Coche implements Vehiculo
{
    public function __construct($marca, $modelo) { /*...*/ }
    public function getData() { /*...*/ }
    public function setData($data) { /*...*/ }
}

class Bicicleta implements Vehiculo
{
    public function __construct($marca) { /*...*/ }
    public function getData() { /*...*/ }
    public function setData($data) { /*...*/ }
}

class Viaje
{
    private $medio;
    private $data;

    public function __construct(Vehiculo $vehiculo)
    {
        $this->medio = $vehiculo;
        $this->data = $vehiculo->getData();
        /* ... */
    }
}

$coche = new Coche('Tesla', 'Model X');
$bicicleta = new Bicicleta('BMC');
$viaje1 = new Viaje($coche);
$viaje2 = new Viaje($bicicleta);
```

> Aunque tanto ***Coche*** como ***Bicicleta*** implementan la interfaz ***Vehiculo***, sus constructores son diferentes. De hecho, aunque es posible que una interfaz defina el constructor, no suele ser una buena práctica, ya que la instanciación es algo muy específico de cada clase.

Existen varios tipos de inyección de dependencias:

- **Inyección en constructor** (*constructor injection*): es la que hemos visto. Dado que la inyección se produce en el constructor, se trata de una dependencia obligatoria.
- **Inyección en** ***setter*** (*setter injection*): la inyección se produce del mismo modo, pero en un método que no es el constructor. Esta dependencia es opcional, ya que solo se utiliza al invocar ciertas funcionalidades de la clase cliente. Normalmente, el método es un *setter*, ya que su invocación suele cambiar el estado (propiedades) del objeto.
- **Inyección de interfaces** (*interface injection*): puede ser de cualquiera de los dos tipos anteriores, pero en lugar de indicar un tipo de clase, se indica una interfaz, como hemos visto.

Sin embargo, esto representa todavía un problema. En el mundo real, una clase puede tener múltiples dependencias, unas obligatorias, otras opcionales, etc. De este modo, necesitaríamos saber de antemano todos los servicios que va a precisar el cliente, tanto los obligatorios como los opcionales, y proceder a instanciarlos todos antes de crear la instancia de la clase cliente. Y esta gestión manual de las dependencias representa un problema muy complejo en la práctica.

### El contenedor de servicios

Aquí es donde entra en juego el *service container*, que no es más que un mecanismo que permite gestionar las dependencias de forma automática. Este tipo de contenedor suele formar parte de un *framework*, ya que tal característica no existe de forma nativa en el lenguaje *PHP*.

Por suerte, *Laravel* proporciona su propio *service container*.

La idea es centralizar en algún punto los detalles acerca de cómo construir (instanciar) cada clase servicio. Así, desaparece la necesidad de instanciar manualmente los servicios que se van necesitando, ya que es precisamente el *service container* quien se encarga de proporcionar dichas instancias de servicios sobre la marcha.

El *service container* está definido en el archivo ***bootstrap/app.php***. Este objeto es de tipo ***Illuminate\\Foundation\\Application***, y puede obtenerse dicha una instancia del contenedor a través del *helper* `app()`.

Supongamos que tenemos una clase llamada ***Cliente*** que depende de otra llamada ***Servicio***. Deberíamos ser capaces de codificar la clase cliente sin preocuparnos de cómo se construye una instancia de la clase servicio. Simplemente deberíamos obtener esa instancia de la clase servicio desde el *service container*, que se encargaría construirla y proporcionarla.

Para instruir al contenedor de servicios acerca de cómo instanciar cada servicio, usaremos su método `bind()`, pasándole como primer argumento el nombre *fully-qualified* de la clase servicio a registrar, y como segundo argumento una *closure* o función que retornará la instancia de la clase. Esta *closure* tiene como parámetro una instancia del mismo *service container*:

```php
app()->bind(Servicio::class, function($app) {
    return new Servicio;
});
```

Al hacer esto, el *service container* añade esta información a su propiedad ***bindings***, que es un *array* con todos los *bindings* (cómo se construye cada clase).

Una vez creado el *binding*, para obtener (resolver) una instancia de una clase concreta a través del *service container*, este puede crear instancias directamente mediante el método `make()`:

```php
$serv = app()->make(Servicio::class);
```

Veamos un ejemplo completo:

```php
class Servicio { /*...*/ }

class Cliente
{
    public function __construct()
    {
        app()->bind(Servicio::class, function($app) {
            return new Servicio;
        });
        $serv = app()->make(Servicio::class);
        // ...
    }
}
```

Lo cierto es que el ejemplo no tiene mucho sentido, ya que la gracia está en no tener que instruir al contenedor de servicios dentro de la clase cliente. Así, la gran mayoría de los servicios quedarán registrados mediante un *service provider*, que instruirá al contenedor al principio de todo, y en las clases nos olvidaremos de registrar *bindings*.

Por otro lado, el *helper* `resolve()` equivale a `app()->make()`, así como `app()` con el nombre *fully-qualified* de la clase como argumento. Así, asumiendo que un *service provider* ya ha registrado el *binding*, la clase quedaría así:

```php
class Cliente
{
    public function __construct()
    {
        $serv = resolve(Servicio::class);
        // ...
    }
}
```

Todavía no estamos usando la inyección automática. Pero ya falta poco.

### *Binding* de interfaces

Cuando registramos el *binding* de una interfaz, la *closure* debe retornar una instancia de la clase que deseemos, siempre y cuando esa clase implemente la interfaz. Es en este punto donde podemos intercambiar entre una u otra clase (*interface injection*).

### Inyección automática

Para invocar directamente un método o instanciar una clase explícitamente (con `new`), si necesitamos inyectar algun servicio, deberemos utilizar el *service container* explícitamente, mediante `resolve()` o equivalente.

Sin embargo, cuando un método o constructor **es invocado automáticamente por** ***Laravel*** (método de un controlador, *handler* de ruta, *event listener*, *middleware*, etc.), la inyección de dependencias se indica dentro de la lista de parámetros de tal método o constructor, sin utilizar `resolve()` manualmente. Es importante tener en cuenta que estos parámetros inyectados automáticamente deben estar *type-hinted* (se debe indicar el tipo), y deben estar **antes** de los parámetros formales.

Este código:

```php
class Servicio {}

class MiController extends Controller
{
    public function __invoke()
    {
        $serv = resolve(Servicio::class);
        // ...
    }
}
```

Se puede (y se debería) escribir así:

```php
class Servicio {}

class MiController extends Controller
{
    public function __invoke(Servicio $serv)
    {
        // ...
    }
}
```

Por ejemplo, en el *handler* de una ruta:

```php
Route::get('/', function(Servicio $serv) { /*...*/ })
```

Así, las dependencias solo tienen que indicarse en la lista de parámetros para que sean inyectadas automáticamente por el *service container*.

Como hemos comentado, si la llamada al método o instanciación de clase la realizamos directamente, este mecanismo no está disponible:

```php
class Servicio {}

class Cliente {
    public function __construct(Servicio $serv) { /*...*/ }
}

// ...

$cliente = new Cliente;  // ERROR! El constructor espera
                         // un argumento...
```

> A modo de organización, la lógica de negocio se debería incluir en servicios inyectados en los controladores, que quedarían así, libres de lógica de negocio. Cabe añadir que la lógica relacionada con bases de datos se debería pasar al modelo (ver capítulo de bases de datos), fuera del controlador. De este modo, los controladores deberían tener muy poco código (*thin controllers*).

### Resolución de configuración cero

Hay que mencionar que **no siempre es necesario registrar un** ***binding***. Esto se puede hacer con clases que cumplan estos requisitos:

- Su constructor no precisa de argumentos que precisen valores concretos, como números o *strings*. Así, todos los parámetros (si los hay) deben ser dependencias *type-hinted*, que en ningún caso pueden ser interfaces.
- La clase no tiene dependencias, y si las tiene, cumplen todas el requisito anterior.

De todas formas, no tiene sentido que las clases pensadas para actuar como servicios para otras clases cliente esperen valores concretos.

```php
class Servicio {}

Route::get('/', function(Servicio $serv) { /*...*/ })
```

En este ejemplo no es necesario registrar el *binding*.

> El contenedor de servicios evalúa las dependencias de una clase mediante el mecanismo de reflexión de *PHP*, que permite obtener información de la propia clase (incluso puede acceder a los comentarios del código).

### *Binding*

El ejemplo anterior era muy simple, ya que la creación del servicio era sin argumentos, pero es posible que deseemos que la creación de la clase (servicio) incluya algunos valores pasados al constructor, o que exista una interfaz entre las dependencias de esta. En ese caso, debemos crear un *service provider* que, dentro de su método `register()`, enlace (*bind*) esa clase con el modo de crear la instancia.

Los *service providers* tienen una propiedad ***app***, que almacena, precisamente, una referencia a la instancia del *service container*. Por otro lado, como hemos visto, también el *helper* `app()` retorna una instancia del mismo.

Supongamos que hemos desarrollado un servicio ***Creditos***. Dentro del método `register()` de un *service provider*:

```php
$this->app->bind(Creditos::class, function($app) {
    return new Creditos("Valor por defecto");
});
```

En este caso, cada vez que se solicite una inyección, se creará una nueva instancia que pasará al método que la solicite. Sin embargo, podemos hacer, al registrar el *service provider*, en lugar de `bind()`:

```php
$this->app->singleton(Creditos::class, function($app) {
    return new Creditos("Valor por defecto");
});
```

En este caso, solo se creará una instancia la primera vez que se solicite, y esta instancia se utilizará en cada inyección (instancia compartida).

También podemos hacer el *binding* directamente a una instancia **ya existente**:

```php
$ob = new Creditos($valor);
$this->app->instance(Creditos::class, $ob);
```
En este caso, esta misma instancia será retornada en subsecuentes llamadas al contenedor para resolver (se evaluará una sola vez).

Como ejemplo, ya que recibimos una instancia del contenedor (***\$app***) en el *resolver*, podemos resolver subdepencias de la clase que estamos *binding*:

```php
$this->app->bind(Cliente::class, function ($app) {
    return new Cliente($app->make(Servicio::class));
});
```

La *facade* ***App*** (***Illuminate\\Support\\Facades\\App***) es otro modo de acceder al contenedor (`App::bind(...)`).

Por otro lado, también podemos **registrar una interfaz**. En la *closure* deberemos retornar una clase (que por lógica implemente esa interfaz). Podemos decidir dinámicamente (durante el método de registro) qué clase y qué valores de inicialización usaremos. Supongamos que dos clases ***CreditosA*** y ***CreditosB*** implementan la interfaz ***InterfazCreditos***:

```php
$this->app->bind(InterfazCreditos::class, function($app) {
    if(<condición>)
        return new CreditosA("Valor por defecto para A");
    return new CreditosB("Valor por defecto para B");
});
```

Si no hay que pasar argumentos al constructor, y solo consideramos una posibilidad, se podría hacer así:

```php
$this->app->bind(InterfazCreditos::class, Creditos::class);
```

Esto simplemente *binds* una interfaz a una implementación concreta de la misma.

No olvidemos que `::class` es simplemente un *string* con el nombre de la clase *fully qualified*.

## Service providers

Los *service providers* registran todos los componentes de la aplicación: *bindings* del *service container*, *event listeners*, *middleware*, rutas, etc.

En la clave ***providers*** de ***config/app.php*** se indica un *array* con los nombres de los *service providers* que deben cargarse.

### Crear un *service provider*

Se trata en realidad de una clase, que a su vez debe extender la clase ***Illuminate\\Support\\ServiceProvider***. Para crear un proveedor de servicions, en línea de comandos:

```
php artisan make:provider nombreProvider
```

Facilita la creación, aunque también se podría hacer a mano.

Existe, por defecto, un *service provider* vacío llamado ***AppServiceProvider***, que puede utilizarse para registrar elementos para nuestra aplicación. Si necesitamos registrar pocas cosas, es una buena práctica utilizar este proveedor. En caso de que tengamos que registrar una gran cantidad de servicios y/o otras características, es mejor agruparlo adecuadamente en distintos proveedores.

#### Método register()

Si el proveedor tiene el método `register()`, se usara única y exclusivamente para registrar *bindings* en el contenedor de servicios como se ha visto antes, **para nada más**.

Si el proveedor registra *bindings* simples, se pueden utilizar las propiedades ***\$bindings*** y ***\$singletons***:

```php
class UnServiceProvider extends ServiceProvider
{
    public $bindings = [
        UnaInterfaz::class => UnaClase::class,
    ];

    public $singletons = [
        InterfazX::class => ClaseX::class,
        InterfazY::class => ClaseY::class
    ];
}
```

#### Método boot()

Una vez **todos** los métodos `register()` de los *service providers* han sido ejecutados, se ejecutan los métodos `boot()`. Por lo tanto en la ejecución de este último, ya tenemos disponibles todos los servicios del contenedor, con lo que se podrían inyectar en este método los servicios que queramos.

Aquí se registran otras cosas.

### Registrar un *service provider*

Para registrar el proveedor de servicios, debe añadirse al *array* en la clave ***providers*** de ***config/app.php***.

### *Deferred providers*

Si el proveedor **únicamente** registra *bindings* en el contenedor de servicios, se puede posponer su registro hasta el momento en que sea necesario tal *binding*. Esto aumenta el rendimiento de la aplicación, ya que no se cargarán estos proveedores en cada petición.

El proveedor se cargará únicamente si se solicita alguno de los servicios que proporciona.

Para posponer la carga, el proveedor debe implementar la interfaz ***Illuminate\\Contracts\\Support\\DeferrableProvider*** y definir un método `provides()` que debería retornar un *array* con los nombres *fully qualified* de las clases (servicios) que registra.

## Facades

Son una forma de acceder a las distintas utilidades que nos proporcional el *framework*.

A través de las *facades* tenemos acceso a gran parte de la funcionalidad de *Laravel*, a través de unas clases *proxy* (las *facades* en sí) que permiten el acceso, mediante una interfaz de método estático, a distintas funciones (métodos, normalmente no estáticos) de otras clases subyacentes, disponibles en el *service container*.

Es decir, al invocar a la *facade*, tenemos acceso a una funcionalidad que es inyectada en dicha *facade*, y que por lo tanto proviene de una clase (servicio) determinada, sin que nos tengamos que preocupar cómo se produce tal mecanismo.

Existen varias *facades* en *Laravel*, definidas en el *namespace* ***Illuminate\\Support\\Facades***.

Adicionalmente, existe todavía otra forma de acceder a gran parte de la funcionalidad de *Laravel* mediante funciones *helper*.

Como ejemplo, si queremos generar una respuesta *JSON*, podemos hacerlo mediante una *facade*, o mediante un *helper*:

```php
use Illuminate\Support\Facades\Response;

// Facade:
$resp1 = Response::json([/*...*/]);

// Helper function:
$resp2 = response()->json([/*...*/]);
```
