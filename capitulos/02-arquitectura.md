# Arquitectura

> En este apartado se hace referencia a los **servicios** que pueden estar disponibles para la aplicación, y como crearlos y registrarlos. Entre estos servicios se hallan las clases y objetos que se inyectan en otras clases a través del mecanismo de inyección de dependencias (que realiza el llamado *service container*).

## Ciclo de vida de una petición

El punto de entrada es el archivo ***public/index.php***. Este carga la definición del *autoloader* de *Composer* y obtiene una instancia de la aplicación a través de ***bootstrap/app.php***.

Luego, la petición entrante es enviada (dependiendo del tipo de *request*) a uno de los *Kernels*: el *HTTP* (***app/Http/Kernel***) o el de consola (***app/Console/Kernel***).

En el caso del *kernel HTTP*, antes de manejar la petición, define un *array* de *bootstrapers* que realizan una serie de tareas, como configurar el *logging* o el tratamiento de errores, o detectar el entorno. También define una lista de *middleware* por el que la petición tendrá que pasar antes de ser tratada por la aplicación. Este *middleware* trata con la sesión *HTTP*, determina si la aplicación está en modo de mantenimiento, verifica el *token*, etc. Luego, pasará la petición a la aplicación a través de su método `handle()`, que simplemente recibe una *request* y retorna una *response*.

Una de las cosas que hacen los *bootstrapers* es registrar (cargar) los *service providers* de la aplicación. Estos están configurados en ***config/app.h***, en el *array* ***providers***. En este caso, el método `register()` del *kernel* se llama sobre cada uno de ellos, y luego se llama al método `boot()`.

Los *service providers* por defecto están en ***app/Providers***.

Los *service providers* registran (activan, cargan) todos y cada uno de los componentes del framework: base de datos, colas, validación, etc.

Una vez han sido registrados todos los *service providers*, se pasará la petición al *router* para que sea manejada.

## Service container

Para entender este mecanismo, debemos comprender lo que es la dependencia y la inyección de dependencias (*dependency injection*).

### Dependencia y *dependency injection*

Supongamos que una clase ***Pelicula*** utiliza un objeto de otra clase, por ejemplo ***Creditos***:

```php
class Creditos
{
    private $roll;

    function __construct() { $this->roll = "Créditos"; }

    function output() { return $this->roll; }
}

class Pelicula
{
    private $contenido;

    function __construct()
    {
        $creditos = new Creditos;    // aquí está el meollo
        $this->contenido = ["Metraje", $creditos->output()];
    }

    function screen() { var_dump($this->contenido); }
}

$peli = new Pelicula;
$peli->screen();
```

Como vemos, la clase ***Pelicula*** se encarga de construir un nuevo objeto de tipo ***Creditos***. Esto es la dependencia: la clase ***Pelicula*** depende de la clase ***Creditos***. Esto podría llegar a ser un problema. Si por ejemplo tenemos docenas de clases que utilizan un objeto de tipo ***Creditos***, y a la larga cambiamos la forma de crear un objeto de tipo ***Creditos*** (por ejemplo, le añadimos un parámetro obligatorio al constructor), entonces deberemos cambiar todas y cada una de las clases que crean un objeto de esta clase, una por una.

Por lo tanto, deberíamos dejar la creación del objeto fuera de la clase ***Pelicula*** y que nos venga el objeto ya creado desde fuera. Eso es la inyección de dependencias: un mecanismo que inyecta esa dependencia (***Creditos***) dentro de nuestra clase (***Pelicula***), es decir, ese mecanismo, que sabe cómo se crea ese objeto, lo crea y nos lo entrega para que lo podamos usar. Esa clase (***Creditos***) conforma un **servicio**, que utiliza la clase **cliente** ***Pelicula***.

Ahora, suponiendo que exista tal mecanismo, ¿cómo hacemos uso de él? Simplemente añadiendo la inyección de esa dependencia en la lista de parámetros (especificando su tipo, y antes de los parámetros normales), con lo que ya podemos prescindir del `new`. En nuestro ejemplo, reescribiendo el constructor de ***Pelicula*** así:

```php
function __construct(Creditos $creditos)
{
    $this->contenido = ["Metraje", $creditos->output()];
}
```

Normalmente esto se hace en el constructor o en un método *setter* (método que establece el valor de una propiedad de la clase).

Evidentemente, si no disponemos del mecanismo de *dependency injection*, esto no solucionará nada: simplemente se producirá un error, dado que el compilador esperará un argumento que no le estamos pasando al constructor.

### Binding

El *service container* es precisamente el mecanismo que realiza la inyección de dependencias.

El ejemplo anterior era muy simple, ya que la creación del servicio era llanamente sin argumentos, pero es posible que deseemos que la creación de la dependencia incluya algún pase de argumentos. En ese caso, debemos crear un *service provider*. Dentro del método `register()` de este, debemos enlazar (*bind*) esta dependencia con el modo de crear el objeto.

Los *service providers* tienen una propiedad ***app***, que retorna, precisamente, una instancia del *service container*. Por otro lado, el *helper* `app()` de *Laravel* también retorna una instancia del mismo.

```php
$this->app->bind(Creditos::class, function($app) {
    return new Creditos("Valor por defecto");
});
```

En el ejemplo (y subsiguientes), se supone que hemos importado el *namespace* donde está definida ***Creditos***, mediante `use`.

En este caso, cada vez que se solicite una inyección, se creará una nueva instancia que pasará al método que la solicite. Sin embargo, podemos hacer, al registrar el *service provider*, en lugar de `bind()`:

```php
$this->app->singleton(Creditos::class, function($app) {
    return new Creditos("Valor por defecto");
});
```

En este caso, solo se hará una sola instancia, que se utilizará en cada inyección.

También podemos hacer el *binding* directamente a una instancia ya existente:

```php
$ob = new Creditos($valor);
$this->app->instance(Creditos::class, $ob);
```

En este caso, esta misma instancia será devuelta en subsecuentes llamadas al contenedor (se evaluará una sola vez).

También podemos registrar una interfaz. En la función *callback* deberemos retornar una clase (que por lógica implemente esa interfaz). Podemos decidir dinámicamente (durante el método de registro) qué clase y qué valores de inicialización usaremos. Supongamos que dos clases ***CreditosA*** y ***CreditosB*** implementan la interfaz ***InterfazCreditos***:

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

### Resolución

Esto sería resolución automática:

```php
public function metodo(Creditos $creditos)
{
    // En $creditos tenemos la instancia que proporciona el service container
}
```

Pero podemos obtener el servicio manualmente del *service container* con el método `make()` del *service container*, al que le pasaremos el nombre de la clase o interfaz en cuestrión (*fully qualified*):

```php
// Desde el interior de un service provider
$creditos = $this->app->make(Creditos::class)
```

Si no estamos dentro de un *service provider* no tendremos acceso a la propiedad ***app***. Pero disponemos siempre del *helper* `app()`, que retorna también una instancia del *service container*.

```php
$creditos = app()->make(Creditos::class)
// Por brevedad, esto es equivalente:
$creditos = app(Creditos::class)
// También podemos hacerlo así:
$creditos = resolve(Creditos::class)
```

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

Si el proveedor tiene el método `register()`, se usara única y exclusivamente para registrar *bindings* en el contenedor de servicios, para nada más. En caso de que se quieran registrar *bindings* **simples**, se pueden usar las propiedades ***\$bindings*** y ***\$singletons***:

```php
public $bindings = [InterfazCreditos::class => Creditos::class];
public $singletons = [InterfazOtra::class => ClaseOtra::class, InteOtraMas::class => ClasOtraMas::class];
```

En este caso, registramos un *binding* y dos *singletons*.

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
