# Arquitectura

## Ciclo de vida de una petición

El punto de entrada es el archivo ***public/index.php***. Este carga la definición del *autoloader* de *Composer* y obtiene una instancia de la aplicación a través de ***bootstrap/app.php***.

Luego, la petición entrante es enviada (dependiendo del tipo de *request*) a uno de los *Kernels*: el *HTTP* (***app/Http/Kernel***) o el de consola (***app/Console/Kernel***).

En el caso del *kernel HTTP*, antes de manejar la petición, define un *array* de *bootstrapers* que realizan una serie de tareas, como configurar el *logging* o el tratamiento de errores, o detectar el entorno. También define una lista de *middleware* por el que la petición tendrá que pasar antes de ser tratada por la aplicación. Este *middleware* trata con la sesión *HTTP*, determina si la aplicación está en modo de mantenimiento, verifica el *token*, etc. Luego, pasará la petición a la aplicación a través de su método `handle()`, que simplemente recibe una *request* y retorna una *response*.

Una de las cosas que hacen los *bootstrapers* es registrar (cargar) los *service providers* de la aplicación. Estos están configurados en ***config/app.h***, en el *array* ***providers***. En este caso, el método `register()` del *kernel* se llama sobre cada uno de ellos, lo cual registra todos los **servicios** (véase más abajo). Una vez se han ejecutado todos los métodos `register()`, se llama los métodos `boot()`, que realizan otras tareas.

Los *service providers* por defecto están en ***app/Providers***, y registran todos los servicios necesarios: base de datos, colas, validación, etc.

Una vez han sido cargados todos los *service providers*, se pasará la petición al *router* para que sea manejada.

## Service container

> Es posible que se deba leer un poco acerca de las rutas y controladores en el siguiente capítulo antes de abordar la lectura de este apartado.

Para entender este mecanismo, debemos comprender lo que es la dependencia y la inyección de dependencias (*dependency injection*). Se denomina **servicio** a una clase que es utilizada por otra clase (llamada clase **cliente**). Entonces, la clase cliente **depende** de la clase servicio.

Supongamos que tenemos una clase ***Actividad***, que en alguno de sus métodos utiliza una instancia de una clase ***Deporte***, que a su vez utiliza en algún momento una clase de tipo ***Rugby***, que a su vez usa una clase de tipo ***Estadio***, etc. Esto puede acabar representando un problema tremendo. Imaginemos que queremos crear una instancia de ***Actividad***. Es posible que tengamos que inicializar cada uno de estos objetos:

```php
$actividad = new Actividad($p1, $p2, new Deporte($p3, new Rugby($p4, $p5, new Estadio, $p6)));
```

A parte de ser bastante lío, ¿qué pasaría si el constructor de ***Deporte*** cambia en cuanto a los parámetros? Habría que ir por todo el código cambiando todas las sentencias de instanciación. En cambio, si tuviésemos información de cómo se construye una instancia de cada una de estas clases, todo sería más sencillo, pues no habría que especificarlo en el código cada vez. Esto es precisamente lo que hace el *service container*.

La instancia del *service container* se crea en el archivo ***bootstrap/app.php***. Este objeto es de tipo ***Illuminate\Foundation\Application***, y puede obtenerse dicha instancia a través del *helper* `app()`.

Supongamos que tenemos una clase llamada ***Cliente*** que depende de otra llamada ***Servicio***. Deberíamos ser capaces de codificar la clase cliente sin preocuparnos de cómo se construye una instancia de la clase servicio. Simplemente deberíamos obtener esa instancia de la clase servicio del *service container*, que se encargaría él mismo de construirla (pues sabe cómo hacerlo).

Para instruir al contenedor acerca de cómo hacerlo, usaremos el método `bind()` del mismo, pasándole como primer argumento el nombre *fully-qualified* de la clase a registrar, y como segundo argumento una *closure* que retornará la instancia de la clase. Esta *closure* puede indicar como primer parámetro una instancia del mismo *service container*:

```php
app()->bind(Servicio::class, function($app) {
    return new Servicio;
});
```

Al hacer esto, el *service container* añade esta información a su propiedad ***bindings***, que es un *array* con todos los *bindings* (cómo se construye cada clase).

Una vez creado el *binding*, para obtener (resolver) una instancia de una clase concreta a través del *service container*, este posee el método `make()`:

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

Lo cierto es que el ejemplo no tiene mucho sentido, ya que la gracia está en no tener que instruir al contenedor. Así, la gran mayoría de los servicios quedarán registrados mediante un *service provider* (véase más adelante), y en las clases nos olvidaremos de registrar *bindings*.

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

### *Binding* de interfaces

Cuando registramos el *binding* de una interfaz, la *closure* debe retornar una instancia de la clase que deseemos, siempre y cuando esa clase implemente la interfaz. Este mecanismo es muy potente, ya que al depender de una interfaz se puede cambiar el servicio de una clase a otra.

### Inyección automática

Cuando un método o función es llamado automáticamente por *Laravel*, es decir, cuando se ejecuta un método de un controlador, un *handler* de ruta, un *event listener*, un *middleware*, etc., podemos indicar la inyección de dependencias dentro de la lista de parámetros de tal método o función. Estos parámetros deben estar *type-hinted* (se indica el tipo), y deben estar **antes** de los parámetros formales.

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

Se puede (y debe) escribir así:

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

Por ejemplo, el *handler* de una ruta:

```php
Route::get('/', function(Servicio $serv) { /*...*/ })
```

Así, las dependencias solo tienen que indicarse en la lista de parámetros para que sean inyectadas automáticamente por el *service container*.

En cambio, si la llamada al método o función la realizamos directamente, la llamada no se hace a través del contenedor, y este mecanismo no está disponible:

```php
class Servicio {}

class Cliente {
    public function __construct(Servicio $serv) { /*...*/ }
}

// ...

$cliente = new Cliente;  // ERROR! El constructor espera 1 argumento...
```

Las dependencias suelen inyectarse en los clientes a través de su constructor, o, en algunos casos, a través de métodos *setter*.

> A modo de organización, la lógica de negocio se debería incluir en servicios inyectados en los controladores, que quedarían así, libres de lógica de negocio.

### Resolución de configuración cero

Hay que mencionar que **no siempre es necesario registrar un** ***binding***. Esto se puede hacer con clases que cumplan estos requisitos:

- Su constructor no precisa de argumentos que precisen un valor concreto, como números o *strings*. Así, todos los parámetros (si los hay) deben ser dependencias *type-hinted*, que en ningún caso pueden ser interfaces.
- La clase no tiene dependencias, y si las tiene, cumplen todas el requisito anterior.

> El contenedor de servicios evalúa las dependencias de una clase mediante el mecanismo de reflexión de *PHP*, que permite obtener información de la propia clase (incluso puede acceder a los comentarios del código).

```php
class Servicio {}

Route::get('/', function(Servicio $serv) { /*...*/ })
```

En este ejemplo no es necesario registrar el *binding*.

### Binding

El ejemplo anterior era muy simple, ya que la creación del servicio era sin argumentos, pero es posible que deseemos que la creación de la dependencia incluya algunos valores pasados al constructor. En ese caso, debemos crear un *service provider* que, dentro de su método `register()`, enlace (*bind*) esa clase con el modo de crear la instancia.

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

También podemos registrar una interfaz. En la *closure* deberemos retornar una clase (que por lógica implemente esa interfaz). Podemos decidir dinámicamente (durante el método de registro) qué clase y qué valores de inicialización usaremos. Supongamos que dos clases ***CreditosA*** y ***CreditosB*** implementan la interfaz ***InterfazCreditos***:

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
});
```

Esto simplemente *binds* una interfaz a una implementación concreta de la misma.

No olvidemos que `::class` es simplemente un *string* con el nombre de la clase *fully qualified*.

## Service providers

Los *service providers* registran todos los componentes de la aplicación: *bindings* del *service container*, *event listeners*, *middleware*, rutas, etc.

En el *array* ***providers*** de ***config/app.php*** pueden verse todos los que son cargados.

### Crear un *service provider*

Debe extender la clase ***Illuminate\\Support\\ServiceProvider***. Para crear uno, en línea de comandos:

```
php artisan make:provider nombreProvider
```

Facilita la creación, aunque también se podría hacer a mano.

Existe, por defecto, un *service provider* vacío llamado ***AppServiceProvider***  que puede utilizarse para añadir elementos para la aplicación.

#### Método register()

Si el proveedor tiene el método `register()`, se usara única y exclusivamente para registrar *bindings* en el contenedor de servicios como se ha visto antes, **para nada más**.

#### Método boot()

Una vez **todos** los métodos `register()` *service providers* han sido ejecutados, se ejecutan los métodos `boot()`. Por lo tanto en la ejecución de este método ya tenemos disponibles todos los servicios del contenedor.

Aquí se registran otras cosas.

### Registrar un *service provider*

Para registrar el proveedor de servicios, debe añadirse al *array* ***providers*** de ***config/app.php***.

### *Deferred providers*

Si el proveedor **únicamente** registra *bindings* en el contenedor de servicios, se puede posponer su registro hasta el momento en que sea necesario tal *binding*. Esto aumenta el rendimiento de la aplicación, ya que no se cargarán estos proveedores en cada petición.

Para posponer la carga, el proveedor debe implementar la interfaz ***Illuminate\\Contracts\\Support\\DeferrableProvider*** y definir un método `provides()` que debería retornar un *array* con los nombres de las clases que registra.

## Facades

Es una forma de acceder a métodos no estáticos como si fuesen estáticos. Existen varias *facades* definidas en Laravel, y están disponibles en el *service container*. Están definidas en el *namespace* ***Illuminate\\Support\\Facades***.

Existe un buen número de ellas. Por citar algunas, tenemos ***App***, ***Cache***, ***Route***, etc.

Para utilizarlas debemos importarlas primero:

```php
use Illuminate\Support\Facades\Route;
```

## Contracts

Son un conjuntos de interfaces que definen los servicios *core* que proporciona el *framework*. Están en el *namespace* base ***Illuminate\\Contracts***.
