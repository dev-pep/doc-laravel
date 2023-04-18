# Modelos *Eloquent*

El mecanimsmo *Eloquent ORM* (*object-relational mapping*, mapeo entre tipos de una base de datos y tipos de lenguaje de programación orientado a objetos) proporciona los **modelos** de *Laravel*. Cada tabla tiene su correspondiente modelo.

Los modelos se almacenan directamente en ***app***. Hay una forma fácil de crear un modelo, mediante `artisan`:

```
php artisan make:model Coche
```

Si además queremos que proceda a la creación de la migración correspondiente, no es necesario que creemos la migración por separado. Bastará con escribir:

```
php artisan make:model Coche --migration
```

Los modelos se almacenan por defecto en ***app/Models***, quedando en el *namespace* ***App\\Models***.

En cuanto a los nombres de los modelos *Eloquent*, existen algunos **convenios importantes**:

- El nombre de la tabla será el mismo que la clase modelo, pero **en plural** (se añade la letra ***s***) y en *snake case* (todo minúsculas y guiones bajos). En este caso, el modelo ***Coche*** guardará sus registros en la tabla ***coches***. Si el modelo fuese ***CarType***, la tabla sería ***car_types***. Por otro lado, si deseamos asociar el modelo a una tabla con un nombre distinto, hay que definir el nombre de la tabla mediante la propiedad (*protected*) ***\$table*** del modelo, que almacenará el nombre de la tabla.
- Cada tabla tendrá un campo con nombre ***id***, que será la clave primaria. Para que se use otro nombre, se debe definir una propiedad protegida en la clase del modelo llamada ***\$primaryKey***, con el nombre del campo con la clave primaria. También se asume que la clave primaria es un entero que se autoincrementa. Si queremos una clave primaria no autoincremental o no numérica, hay que definir la propiedad pública ***\$incrementing*** a ***false***. Además, si el tipo no es numérico, habría que definir la propiedad protegida ***\$keyType*** con un *string* con el valor ***string***.
- Cada tabla tendrá las columnas ***created_at*** y ***updated_at***, que actualizará *Eloquent* automáticamente. Si deseamos que no lo haga, se incluirá la propiedad pública ***\$timestamps*** con valor ***false***. Si queremos cambiar los nombres de los campos donde se guardarán los *timestamps*, se deben definir en la clase las constantes *CREATED_AT* y/o *UPDATED_AT*, conteniendo *strings* con los nombres deseados.

Los modelos *Eloquent* necesitan obligatoriamente una clave primaria, y no admiten claves primarias compuestas (formadas por varios campos). Aunque sí se pueden crear índices compuestos.

Si deseamos asociar el modelo a una conexión de base de datos distinta de la conexión por defecto, deberemos incluir una propiedad protegida ***\$connection*** conteniendo un *string* con el nombre de la conexión deseada.

Si queremos dar un valor por defecto a alguno de los campos, definiremos la variable protegida ***\$attributes***. Se le dará como valor un *array* con los nombres de los atributos a los que queramos dar valor por defecto, junto con sus valores:

```php
protected $attributes = [
    'delayed' => false,
    'city' => 'Sabadell'
];
```

## Acceso a la base de datos

Una vez tenemos nuestro modelo y su tabla correspondiente en la base de datos, podemos leer el contenido entero de la tabla:

```php
$coches = Coche::all();
```

El método `all()` retorna una *collection* de tipo ***Illuminate\\Database\\Eloquent\\Collection***, que extiende el tipo colección de *Laravel*, y cuyos elementos son modelos *Eloquent* (en el ejemplo, de tipo ***App\\Models\\Coche***).

Estas *collections* de *Eloquent* aceptan métodos análogos a los de los *query builders*, los cuales retornan a su vez una *collection Eloquent*, de tal modo que se pueden ir encadenando métodos, al igual que sucedía con los *query builders*. Por lo tanto, en el caso de las colecciones *Eloquent* no hay que llamar a `get()` al final de la cadena.

En cambio, si el primer método llamado es el correspondiente a un *query builder* (como `where()`, `select()`, etc.), no retornará una *collection*, sino un *query builder Eloquent* (basado en un *query builder* convencional), de tipo ***Illuminate\\Database\\Eloquent\\Builder***. En tal caso, podríamos ir refinando la *query*, y al final sí deberíamos usar `get()` (que retornará una colección):

```php
// Las dos sentencias son equivalentes:
$coches1 = Coche::all()
    ->where('id', '>', '5');
$coches2 = Coche::where('id', '>', '5')
    ->get();
```

Si se aplica, por ejemplo, `where()` a un *builder*, este retorna un *builder*. Si se aplica a una *collection*, retorna una *collection*. En otros casos, como `select()`, solo está disponible para *builders*.

> Es posible pasar de colección *Eloquent* a *query builder Eloquent* mediante el método `toQuery()` de las colecciones *Eloquent*. El *query builder Eloquent* retornado tendrá una restricción ***whereIn*** para que se forme el subconjunto de registros adecuado. Es la operación inversa al método `get()` de las colecciones *Eloquent*.
>
> Las colecciones *Eloquent* disponen también del método `toArray()` que convierte dicha colección en un simple *array* no asociativo.

Si como primer método indicamos `find()` con un *array* de claves primarias, recibiremos, de forma similar a `all()`, una **colección** de registros coincidentes (si los hay), no un *builder*.

En el caso de los métodos que retornan un solo registro (como `find()` con un solo valor, no *array*, o `first()`), el tipo retornado es el del modelo (en este caso ***App\\Models\\Coche***). Estos métodos están disponibles tanto en colecciones como en *builders*, así como en el mismo modelo (por lo que pueden estar en primer lugar: `Coche::first()`).

Cuando el conjunto de registros retornados es cero, por ejemplo cuando `find()` con un argumento no encuentre la clave primara, lo retornado es ***null***.

> Podemos definir tantos métodos como queramos en el modelo. De hecho, toda la lógica relacionada con la base de datos debería estar definida en los modelos.
>
> Los nombres de los campos de la tabla pueden accederse como propiedades del objeto con el mismo nombre. Desde el código del modelo el acceso se puede hacer como `$this->campo`.

## Refresco

Para refrescar el valor de un modelo, es decir, obtener su valor actual desde la base de datos, se puede usar el método `fresh()`. Este método no cambia el valor del modelo.

```php
$coche = Coche::where('marca', 'Citroen')->first();
// Cambios, etc.
$cocheFresh = $coche->fresh();
```

En cambio, el método `refresh()` sí actualiza el valor actual del modelo con los valores actuales en la base de datos.

## Fragmentos

Las colecciones y *builders* *Eloquent* pueden fragmentarse mediante el método `chunk()` o `chunkById()`, de forma similar a como se hacía con los métodos del mismo nombre en la *facade* ***DB***.

## Lectura de modelos o agregados

Para leer un modelo, como hemos visto, se pueden usar los métodos `find()` con argumento no *array*, o `first()`. Se puede combinar este último con un `where()` mediante el método `firstWhere()`:

```php
$coche = Coche::where('marca', 'Citroen')->first();
// Equivalente:
$coche = Coche::firstWhere('marca', 'Citroen');
```

También existe el método `firstOr()`, el cual recibe una *closure* que se ejecutará solamente si no existen resultados en la consulta. El valor que retornará el método será lo que retorne esa *closure*.

Por otro lado, los métodos `findOrFail()` y `firstOrFail()` levantarán una excepción si no hay resultados en la consulta. Si la excepción no se captura, se retornará al cliente un código de respuesta 404.

Otra aproximación es la del método `firstOrCreate()`. En este caso, el primer argumento es un *array* con pares clave/valor que se utilizará para localizar un registro en la tabla. Si existe algún registro coincidente, el valor retornado será el modelo con los datos del primer registro con estos valores. En caso de no encontrarlo, se creará un nuevo registro en la base de datos cuyos campos serán los de este primer argumento, mezclado con los valores indicados en el segundo argumento (otro *array* con valores para los campos). El valor retornado será el de este modelo con los datos encontrados, o con los datos del nuevo registro.

El método `firstOrNew()` hace lo mismo que `firstOrCreate()`, con la diferencia que en caso de no encontrar el registro indicado, el nuevo modelo creado no es persistido en la base de datos. Para hacerlo, se deberá indicar explícitamente más tarde (método `save()`, como veremos).

En lugar de modelos es posible obtener valores agregados mediante métodos como `count()`, `sum()`, `max()`, etc. Estos métodos retornan un valor escalar, no un modelo.

## Insertar registros

Para añadir un registro, se crea una nueva instancia del modelo, se le dan los valores pertinentes, y se invoca el método `save()` del mismo.

```php
$coche = new Coche;
$coche->name = $req->name;  // etc.
$coche->save();
```

### Inserción masiva

Es posible insertar un registro en la base de datos a través del modelo, en una sola línea, usando el método `create()`. Por defecto, la inserción masiva está deshabilitada por representar un potencial problema de seguridad.

## Modificar registros

Si queremos modificar un registro, usaremos también el método `save()`, pero en este caso, la clave primaria del modelo ya existe en la base de datos:

```php
$coche = Coche::find(1);
$coche->name = $req->name;  // etc.
$coche->save();
```

### Modificación masiva

Es posible modificar una colección completa de modelos, mediante el método `update()`:

```php
Coche::where('marca', 'Tesla')
    ->update(['electrico' => 1]);
```

Dicho método acepta un *array* de campos con los respectivos valores que deben tener en la base de datos. Retorna el número de registros afectados.

## Examinar cambios

El método `isDirty()` del modelo retorna un booleano que indica si dicho modelo ha sido modificado desde que se leyó de la base de datos. Es el inverso de `isClean()`. Si a estos métodos se les pasa un *string* con un nombre de campo o un *array* con varios nombres de campo, la comprobación se reduce al campo o campos indicados.

De forma similar, el método `wasChanged()` indica si el modelo ha sido grabado en la base de datos con valores distintos a los que tenía cuando se leyó de dicha base de datos. La comprobación también puede reducirse a los campos deseados.

El método `getOriginal()` retorna un *array* con los valores de los campos en el momento que se leyó de la base de datos, independientemente de los cambios que haya sufrido. Si se le pasa el nombre de un campo, retornará el valor que tenía ese campo.

## *Upserts*

El método `updateOrCreate()` funciona de forma análoga a `firstOrCreate()`. La diferencia es que si el registro es encontrado (primer *array*), se actualiza con los valores indicados (segundo *array*).

Si se quieren hacer varios de estos *upserts* de un golpe, existe el método `upsert()`, que funciona de forma análoga al método de mismo nombre de los *query builders* de la *facade* ***DB***.

## Eliminación de registros

Para eliminar un registro, usaremos el método `delete()` sobre un solo elemento (instancia del modelo), sobre una colección *Eloquent* o sobre un *query builder Eloquent*.

```php
$coche = Coche::find(['1', '2', '3']);
$coche->delete();
```

También es posible eliminar todos los registros de la tabla asociada al modelo mediante el método `truncate()` (sin argumentos).

Si conocemos las claves primarias de los registros a eliminar, podemos usar `destroy()` sobre una o varias claves primarias, un *array* o una colección de las mismas:

```php
Coche::destroy(1, 2, 3);
Coche::destroy(25);
Coche::destroy([21, 22, 23]);
Coche::destroy(collect([21, 22, 23]));
```

## Comparación de modelos

Para saber si un modelo concreto es el mismo que otro, es decir, si ambos tienen la misma conexión de base de datos y corresponden a la misma clave primaria en la misma tabla, tenemos los métodos `is()` e `isNot()`:

```php
if($coche1->is($coche2))
    // ...
if($coche1->isNot($coche2))
    // ...
```

## Relaciones

*Eloquent* nos permite definir el tipo de relaciones en una base de datos relacional. Veremos aquí cómo definir algunos de estos tipos y qué ventajas ofrece.

### Relación uno a uno

Supongamos una base de datos en la que tenemos una tabla de departamentos de una empresa (modelo ***Departamento***), y una tabla de jefes de departamento (modelo ***Jefe***). Cada departamento tiene un solo jefe y cada jefe lo es de un solo departamento. Para definir esta relación uno a uno, podríamos hacer lo siguiente en el modelo ***Departamento***:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Departamento extends Model
{
    public function jefe() {
        return $this->hasOne(Jefe::class);
    }
}
```

Esto indica que a cada departamento le corresponde un jefe. Una vez hecho esto, se puede acceder a dicho jefe a través del modelo de departamento, utilizando el nombre del método que hemos definido, como un atributo dinámico:

```php
$departamento = Departamento::find(33);
$jefeDep = $departamento->jefe;
```

Internamente, *Eloquent* localiza al jefe a través del convenio de claves externas: en este caso, asumirá que en el modelo ***Jefe*** (tabla ***jefes***) existirá un campo (clave externa) ***departamento_id*** cuyo valor será igual al campo ***id*** (clave primaria) del modelo actual, es decir ***Departamento*** (tabla ***departamentos***).

Si el nombre del campo correspondiente a la clave externa no fuese ***departamento_id***, tendríamos que informar de ello a la hora de crear la relación:

```php
return $this->hasOne(Jefe::class, 'depar_ident');
```

Del mismo modo, si el nombre de nuestra clave primaria no es ***id***, también hay que informar de ello en el tercer argumento a `hasOne()`:

```php
return $this->hasOne(Jefe::class, 'depar_ident', 'ident');
```

Así, podemos acceder al jefe a partir del departamento. Para poder hacerlo a la inversa (acceder al departamento a partir del jefe), debemos definir dentro del modelo ***Jefe***:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Jefe extends Model
{
    public function depar() {
        return $this->belongsTo(Departamento::class);
    }
}
```

Ahora ya podríamos hacer:

```php
$jefeDep = Jefe::find(3333);
$departamento = $jefeDep->depar;
```

En este caso, *Eloquent* asume una clave primaria con nombre ***id*** en el modelo ***Departamento*** (tabla ***departamentos***), y una clave externa ***departamento_id*** en el modelo actual ***Jefe*** (tabla ***jefes***). Si los nombres de los campos difieren, nuevamente hay que informar: primero el nombre del campo con la clave externa en nuestro modelo (***Jefe***, tabla ***jefes***), y, si procede, el nombre de la clave primaria en el modelo externo (***Departamento***, tabla ***departamentos***):

```php
return $this->belongsTo(Departamento::class, 'depar_ident', 'ident');
```

### Relación uno a muchos

En la relación uno a muchos, supongamos que cada departamento tiene una serie de empleados (modelo ***Empleado***), y que un empleado solo puede trabajar en un departamento. Entonces, haríamos lo siguiente:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Departamento extends Model
{
    public function empleados() {
        return $this->hasMany(Empleado::class);
    }
}
```

Podríamos hacer posteriormente:

```php
$departamento = Departamento::find(33);
$emps = $departamento->empleados;
```

En este caso, lo retornado no será un modelo empleado, sino una colección *Eloquent* (de tipo ***Illuminate\\Database\\Eloquent\\Collection***) con todos los empleados que trabajan en ese departamento. Como ya hemos visto, una colección *Eloquent* admite métodos análogos a los de un *query builder*, con lo que puede refinarse la relación de registros.

En este caso, *Eloquent* asume que el modelo ***Empleado*** (tabla ***empleados***) contiene un campo ***departamento_id***, que es la clave externa que debe coincidir con la clave primaria ***id*** del modelo actual ***Departamento*** (tabla ***departamentos***). En este caso, podrá haber más de un registro de ***empleados*** con el mismo valor en su clave externa ***departamento_id***.

Como en el caso de las relaciones uno a uno, si los nombres de los campos no son los estándar, hay que indicarlo a la hora de definir la relación:

```php
return $this->hasMany(Empleado::class, 'depar_ident');
```

O:

```php
return $this->hasMany(Empleado::class, 'depar_ident', 'ident');
```

Podemos así acceder a la lista de empleados a partir del departamento. Para hacerlo a la inversa (acceder al departamento a partir de un empleado), debemos definir dentro del modelo ***Empleado***:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Empleado extends Model
{
    public function depar() {
        return $this->belongsTo(Departamento::class);
    }
}
```

Ahora ya podríamos hacer:

```php
$emp = Empleado::find(3333);
$departamento = $emp->depar;
```

Esto retorna el modelo correspondiente al departamento en el que trabaja el empleado (retorna un solo registro, no una colección).

Como siempre, *Eloquent* asume una clave primaria con nombre ***id*** en la tabla de departamentos, y una clave externa ***departamento_id*** en nuestra tabla de empleados. Si hay divergencias en los nombres, nuevamente hay que informar de ello: primero el nombre de la clave externa en nuestro modelo, y, si procede, el nombre de la clave primaria en el modelo externo:

```php
return $this->hasOne(Jefe::class, 'depar_ident', 'ident');
```

### Relación muchos a muchos

Una relación muchos a muchos necesita de tres tablas. Siguiendo con el ejemplo anterior, supongamos que un empleado puede trabajar en más de un departamento. En este caso, la relación entre empleados y departamentos es muchos a muchos.

Las tablas involucradas en la base de datos serían, en primer lugar, ***empleados*** y ***departamentos***. Pero en este caso, no puede haber una clave externa en ***empleados*** llamada ***departamento_id***, puesto que puede haber varios departamentos asociados al empleado. Del mismo modo, tampoco podría haber una clave externa en ***departamentos*** llamada ***empleado_id***, puesto que eso solo permitiría un empleado por departamento.

Así, que la relación debe apoyarse en una tercera tabla que realice todas las asociaciones. *Eloquent* asume que el nombre de esta tabla intermedia estará compuesto por los nombres de las dos tablas involucradas, en singular (sin la ***s*** final), en orden alfabético, y en *snake case*. En este caso, el resultado sería la tabla ***departamento_empleado***.

El convenio que toma *Eloquent* es el de considerar que esta nueva tabla tiene dos únicos campos: cada uno de ellos es una clave externa correspondiente a cada una de las dos tablas a relacionar. En este caso, los dos campos serían ***departamento_id*** (asociado al campo ***id*** de la tabla ***departamentos***) y ***empleado_id*** (asociado al campo ***id*** de la tabla ***empleados***).

Esta nueva tabla es la que relaciona todos los empleados con todos los departamentos en los que trabajan. Con esta estructura en mente, ya podríamos definir la relación:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Empleado extends Model
{
    public function depars() {
        return $this->belongsToMany(Departamento::class);
    }
}
```

Para acceder a los departamentos de un empleado, conseguimos la colección *Eloquent* de la forma habitual:

```php
$emp = Empleado::find(3333);
$departamentos = $emp->depars;
```

Como siempre, se puede refinar el conjunto de elementos de la colección mediante los métodos de *query builder*.

Si la tabla de relaciones tiene un nombre distinto al esperado (normalmente sería ***departamento_empleado***), se debe indicar su nombre a la hora de definir la relación:

```php
return $this->belongsToMany(Departamento::class, 'emple_depar');
```

Además podemos indicar los nombres de las claves externas en esa tabla, si no siguen el estándar:

```php
return $this->belongsToMany(Departamento::class, 'emple_depar', 'emple_id', 'depar_id');
```

En este caso, el tercer argumento es el nombre de la clave externa en el modelo desde el que estamos definiendo el método, mientras que el cuarto argumento es el nombre de la clave externa en el modelo con el que nos estamos relacionando.

Para definir la contraparte, es decir, para poder acceder a todos los empleados de un departamento, la acción es exactamente igual, pero invirtiendo los modelos:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Departamento extends Model
{
    public function empleados() {
        return $this->belongsToMany(Empleado::class);
    }
}
```

#### Acceso a la tabla intermedia

La tabla intermedia de relaciones se utiliza internamente por parte de *Eloquent* para vincular dos tablas mediante una relación muchos a muchos. Pero ¿cómo se podría acceder a los registros de esa tabla directamente? *Eloquent* permite hacerlo mediante el atributo ***pivot*** de los modelos de las dos tablas relacionadas. Siguiendo con el ejemplo anterior:

```php
$emp = Empleado::find(3333);
foreach($emp->depars as $dep) {
    $v1 = $emp->pivot->empleado_id;
    $v2 = $emp->pivot->departamento_id;
}
```

Como vemos, en la colección retornada por ***\$emp->depars***, se ha añadido automáticamente a cada elemento un atributo ***pivot*** que es en realidad un modelo de la tabla intermedia.

Por defecto, se puede acceder únicamente a los dos campos con las claves externas de la tabla intermedia. En el ejemplo, la tabla intermedia (***departamento_empleado***) contenía únicamente esos dos campos (***empleados_id*** y ***departamentos_id***).

Pero, ¿qué pasa si para cada relación queremos añadir información adicional? Supongamos que para cada relación empleado-departamento deseamos añadir la fecha en la que el empleado se incorporó al departamento en cuestión, así como la fecha de baja. En este caso, la tabla intermedia tendrá, por ejemplo, los campos adicionales ***fecha_incorp*** y ***fecha_baja***, a parte de las dos claves externas.

Para que se pueda acceder a estos campos extra, es necesario informar a *Eloquent* a la hora de definir la relación, pasando sus nombres como argumento al método `withPivot()`. Adicionalmente, si deseamos que *Eloquent* gestione también los *timestamps* de la tabla intermedia al acceder a través del atributo ***depars***, añadiremos el método `withTimestamps()`:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Empleado extends Model
{
    public function depars() {
        return $this->belongsToMany(Departamento::class)
            ->withPivot('fecha_incorp', 'fecha_baja')
            ->withTimestamps();
    }
}
```

Ahora ya podríamos acceder a esa información adicional:

```php
$emp = Empleado::find(3333);
foreach($emp->depars as $dep) {
    $incor = $emp->pivot->fecha_incorp;
    $baja = $emp->pivot->fecha_baja;
}
```

#### Retorno de un modelo con relaciones

Si retornamos un modelo (o colección de modelos), no se incluirán las relaciones en una posible respuesta, a no ser que se haga una referencia a las mismas antes de retornarlas. Dicha referencia puede hacerse dentro de la lógica del controlador, o también en la misma plantilla *Blade*.

Pero si por ejemplo retornamos simplemente una respuesta *JSON*, habrá que realizar una referencia explícita antes de retornar.

Así, la primera vez que se realiza una referencia a la relación, *Eloquent* da valor a la propiedad adecuada; cualquier cambio que se realice en esta propiedad se conservará hasta el final.

### Modelo por defecto

Es posible que una clave externa tenga valor ***null***. En ese caso, es posible establecer un modelo por defecto que será retornado en estas circunstancias por funciones como `hasOne()` o `belongsTo()` (funciones que retornan un modelo, no una colección). Ello se consigue pasando como argumento al método `withDefault()` un *array* asociativo con los valores de los campos del modelo deseado:

```php
return $this->belongsTo(Departamento::class)
    ->withDefault([
        'nombre' => 'Compras',
        'planta' => 3
]);
```

## Accessors, mutators y casting

Los *accessors*, *mutators* y *casting* son un mecanismo para modificar los valores de un modelo al leer o escribir.

### Accessors

Un *accessor* modifica un atributo *Eloquent* al ser leído. Para crearlo, hay que escribir un método protegido en la clase del modelo. El nombre del método, en *camel case*, debe corresponderse al nombre del campo al que representa (en *snake case*).

El método *accessor* de un modelo ***Usuario*** se definirá de este modo:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class Usuario extends Model
{
    protected function primerApellido(): Attribute
    {
        return Attribute::make(
            get: fn($valor) => ucfirst($valor)
        );
    }
}
```

En este caso, estamos definiendo un *accessor* para el campo ***primer_apellido*** del modelo ***Usuario***. El método `make()` de la clase ***Atribute*** tiene un parámetro con nombre ***get*** al que se le pasa una *closure*, en este caso una función *arrow*. El argumento que recibe dicha función es el valor leído de la base de datos, sin modificar, del campo ***primer_apellido***. Lo que retorna la *closure* es el valor modificado, en este caso, el apellido con el primer carácter en mayúsculas. Por lo tanto, al acceder a este campo desde el modelo, obtendremos este valor modificado.

### Mutators

Un *mutator* es la inversa de un *accessor*, es decir, será una modificación realizada sobre un valor en el momento de ser escrito en la base de datos. El mecanismo es similar al de un *accessor*, pero en este caso, es el parámetro ***set*** del constructor del objeto ***Attribute*** lo que hay que definir (una vez más, mediante una *closure*). Veamos un ejemplo, en el que se definte tanto el *accessor* como el *mutator*:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class Usuario extends Model
{
    protected function primerApellido(): Attribute
    {
        return Attribute::make(
            get: fn($valor) => ucfirst($valor),
            set: fn($valor) => strtolower($valor)
        );
    }
}
```

La *closure* del *mutator* recibe como valor el valor que se envía al modelo para ser escrito a la base de datos. Sin embargo, este valor será modificado y retornado por la *closure*, de tal modo que este valor modificado es el que llegará a escribirse en la base de datos.

En el ejemplo, el apellido se lee con el primer carácter en mayúscula, mientras que siempre se almacenará en minúsculas.

### Versiones anteriores

En versiones anteriores de *Laravel*, que trabajaba con lenguaje anterior a *PHP* 8, no se podían especificar argumentos con nombre. Entonces, los mecanismos para crear *accessors* y *mutators* se limitaban a definir métodos públicos *getters* y *setters* para los campos deseados, en los que el nombre del método empezaba por ***get*** o ***set***, terminaba en ***Attribute*** y contenía el nombre del campo en *pascal case*:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Usuario extends Model
{
    public function getPrimerApellidoAttribute($valor) {
        return ucfirst($valor);
    }

    public function setPrimerApellidoAttribute($valor) {
        $this->attributes['primer_apellido'] = strtolower($valor);
    }
}
```

### Casting

El mecanismo de *casting* es similar al de *accessors* y *mutators*, pero está enfocado en la conversión de tipos entre *PHP* y la base de datos.

En la propiedad ***\$casts*** del modelo podemos definir mediante un *array* el tipo al que se mapeará cada campo concreto de la tabla. Las claves del *array* son los nombres de los campos y los valores son los tipos *PHP*. Estos son algunos de los tipos que existen por defecto: ***integer***, ***real***, ***float***, ***double***, ***decimal:N***, ***string***, ***boolean***, ***object***, ***array***, ***collection***, ***date***, ***datetime*** y ***timestamp***.

En el caso de ***decimal*** se debe indicar el número de dígitos (p.e. ***decimal:5***).

```php
protected $casts = [
    'motor_electrico' => 'boolean',
    'marca' => 'string',
    'marca' => 'string',
    'km' => 'integer',
    'matriculacion' => 'date'
];
```

A parte de los *casts* por defecto, se puede definir un *cast* a medida. Para ello debemos escribir una clase que implemente la interfaz ***Illuminate\\Contracts\\Database\\Eloquent\\CastsAttributes***. La clase debe definir las conversiones mediante el método `get()` (de base de datos a objeto *PHP*) y `set()` de objeto a base de datos. Luego, a la hora de definir el mapeo de un campo en el *array* ***\$casts***, se indicará como valor el nombre *fully qualified* de la clase.

Tanto el método `get()` como el `set()` reciben como argumentos: una instancia del modelo, un *string* con el nombre del campo, otro con el valor del mismo, y finalmente un *array* con atributos. Cada método retorna el valor del tipo adecuado (traducido).
