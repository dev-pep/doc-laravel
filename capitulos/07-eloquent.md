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

- El nombre de la tabla será el mismo que la clase modelo, pero **en plural** (se añade la letra ***s***) y en ***snake case*** (todo minúsculas y guiones bajos). En este caso, el modelo ***Coche*** guardará sus registros en la tabla ***coches***. Si el modelo fuese ***CarType***, la tabla sería ***car_types***. Por otro lado, si deseamos asociar el modelo a una tabla con un nombre distinto, hay que definir el nombre de la tabla mediante la propiedad (*protected*) ***\$table*** del modelo, que almacenará el nombre de la tabla.
- Cada tabla tendrá un campo con nombre ***id***, que será la clave primaria. Para que se use otro nombre, se debe definir una propiedad protegida en la clase del modelo llamada ***\$primaryKey***, con el nombre del campo con la clave primaria. También se asume que la clave primaria es un entero que se autoincrementa. Si queremos una clave primaria no autoincremental o no numérica, hay que definir la propiedad pública ***\$incrementing*** a ***false***. Además, si el tipo no es numérico, habría que definir la propiedad protegida ***\$keyType*** con un *string* con el valor ***string***.
- Cada tabla tendrá las columnas ***created_at*** y ***updated_at***, que actualizará *Eloquent* automáticamente. Si deseamos que no lo haga, se incluirá la propiedad pública ***\$timestamps*** con valor ***false***. Si queremos cambiar los nombres de los campos donde se guardarán los *timestamps*, se deben definir en la clase las constantes *CREATED_AT* y/o *UPDATED_AT*, conteniendo *strings* con los nombres deseados.

Los modelos *Eloquent* necesitan obligatoriamente una clave primaria, y no admiten claves primarias compuestas (formadas por varios campos). Aunque sí se pueden crear índices compuestos.

Si deseamos asociar el modelo a una conexión de base de datos distinta de la conexión por defecto, deberemos incluir una propiedad protegida ***\$connection*** conteniendo un *string* con el nombre de la conexión deseada.

Si queremos dar un valor por defecto a alguno de los campos, definiremos la variable protegida ***\$attributes***. Se le dará como valor un *array* con los nombres de los atributos a los que queramos dar valor por defecto, junto con sus valores:

```php
protected $attributes = [ 'delayed' => false, 'city' => 'Sabadell' ];
```

## Acceso a la base de datos

Una vez tenemos nuestro modelo y su tabla correspondiente en la base de datos, podemos leer el contenido entero de la tabla:

```php
$coches = App\Coche::all();
```

El método `all()` retorna una *collection* de tipo ***Illuminate\\Database\\Eloquent\\Collection***, que extiende el tipo colección de *Laravel*, y cuyos elementos son modelos *Eloquent* (en el ejemplo, de tipo ***App\\Coche***).

Estas *collections* de *Eloquent* aceptan métodos análogos a los de los *query builders*, los cuales retornan a su vez una *collection Eloquent*, de tal modo que se pueden ir encadenando métodos, al igual que sucedía con los *query builders*. Por lo tanto, en el caso de las colecciones *Eloquent* no hay que llamar a `get()` al final de la cadena.

En cambio, si el primer método llamado es el correspondiente a un *query builder* (como `where()`, `select()`, etc.), no retornará una *collection*, sino un *query builder Eloquent* (basado en un *query builder* convencional), de tipo ***Illuminate\\Database\\Eloquent\\Builder***. En tal caso, podríamos ir refinando la *query*, y al final sí deberíamos usar `get()` (que retornará una colección):

```php
// Las dos sentencias son equivalentes:
$coches1 = App\Coche::all()
    ->where('id', '>', '5');
$coches2 = App\Coche::where('id', '>', '5')
    ->get();
```

Si se aplica, por ejemplo, `where()` a un *builder*, este retorna un *builder*. Si se aplica a una *collection*, retorna una *collection*. En otros casos, como `select()`, solo está disponible para *builders*.

Para pasar de colección a *builder* y viceversa, disponemos de los métodos `toQuery()` (de las colecciones) y `fromQuery()` (de las *queries*). Hay métodos que cambian de uno a otro automáticamente.

Si como primer método indicamos `find()` con un *array* de claves primarias, recibiremos, de forma similar a `all()`, una **colección** de registros coincidentes (si los hay), no un *builder*.

En el caso de los métodos que retornan un solo registro (como `find()` con un solo valor, no *array*, o `first()`), el tipo retornado es el del modelo (en este caso ***App\\Coche***). Estos métodos están disponibles tanto en colecciones como en *builders*, así como en el mismo modelo (por lo que pueden estar en primer lugar: `Coche::first()`).

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

Otra aproximación es la del método `firstOrCreate()`. En este caso, el primer argumento es un *array* con pares clave/valor que se utilizará para localizar un registro en la tabla. Si existe alguno, el valor retornado será el modelo con los datos del primer registro con estos valores. En caso de no encontrarlo, se creará un nuevo registro en la base de datos cuyos campos serán los de este primer argumento, mezclado con los valores indicados en el segundo argumento (otro *array* con valores para los campos). El valor retornado será el de este modelo con los datos encontrados, o con los datos del nuevo registro.

El método `firstOrNew()` hace lo mismo que `firstOrCreate()`, con la diferencia que en caso de no encontrar el registro indicado, el nuevo modelo creado no es persistido en la base de datos. Para hacerlo, se deberá indicar explícitamente más tarde (método `save()`, como veremos).

En lugar de modelos es posible obtener valores agregados mediante métodos como `count()`, `sum()`, `max()`, etc. Estos métodos retornan un valor escalar, no un modelo.




## Insertar o modificar registros

Para añadir un registro, se crea una nueva instancia del modelo, se le dan los valores pertinentes, y se invoca el método `save()` del mismo.

```php
$coche = new Coche;
$coche->name = $req->name;  // por ejemplo
$coche->save();
```

Si queremos modificar un registro:

```php
$coche = App\Coche::find(1);
$coche->name = $req->name;  // por ejemplo
$coche->save();
```

Para inserciones y/o modificaciones, se pueden usar los métodos de *query builder* del modelo (`insert()`, `update()`, etc.).

## Eliminación de registros

Para eliminar un registro, usaremos el método `delete()` sobre un solo elemento, sobre una colección o sobre un *query builder*.

```php
$coche = App\Coche::find(['1', '2', '3']);
$coche->delete();
```

Si conocemos las claves primarias de los registros a eliminar, podemos usar `destroy()`:

```php
App\Coche::destroy(1, 2, 3);
App\Coche::destroy(25);
App\Coche::destroy([21, 22, 23]);
```

## *Mutators*

Este mecanismo permite gestionar el acceso a los atributos de un modelo *Eloquent*.

### *Accessors*

Un *accessor* es un método definido dentro de la clase del modelo que define el acceso a un atributo concreto. El formato del nombre de este método debe ser del tipo ***get + NombreDelCampo + Attribute***, y se refiere a una columna de la tabla cuyo nombre es del tipo ***nombre_del_campo***. Obsérvese la disposición de mayúsculas y guiones bajos. Por ejemplo, el método ***getCilindradaMotorAttribute()*** se referiría al atributo (campo) del modelo ***cilindrada_motor***. El método debe retornar el valor del campo.

Si el campo ya existe en la tabla, este método permite manipular su valor, recibiendo el valor original en la tabla como primer parámetro:

```php
public function getCilindradaMotorAttribute($valor)
{
    return $valor . ' c.c.';
}
```

En cambio si no existe tal campo en la tabla, no recibe ningún valor, y permite retornar algún tipo de campo calculado, normalmente a partir del valor de otros campos del registro. Se puede acceder a ellos a través de `$this->atributo`.

### *Mutators*

Un *mutator* tiene el mismo formato de nombre que un *accessor*, pero en lugar de retornar un valor, establece un valor de atributo. La forma correcta de hacerlo, es dando valor al elemento correspondiente de la propiedad ***\$attributes***:

```php
public function getMarcaAttribute($valor)
{
    $this->attributes['marca'] = strtolower($valor);
}
```

En este caso, suponiendo que ***\$registro*** contenga un modelo de ***Coche***, al hacer ```$registro->marca = 'BMW'```, el valor del atributo ***marca*** será ***bmw***, ya que el valor pasará a través del *mutator*, que lo pasa a minúsculas.

### *Mutators* de fechas

Por defecto, *Eloquent* traduce los campos ***created_at*** y ***updated_at*** a tipo ***Carbon***, que es una extensión del ***DateTime*** de *PHP*. Si tenemos otros campos de fecha que deseamos que sean mapeados a tipo ***Carbon***, debemos añadir sus nombres al array ***\$dates***:

```php
protected $dates = ['fecha_inicio', 'fecha_fin'];
```

Cuando es así, podemos dar valor a un campo fecha del modelo usando un *timestamp UNIX*, un *date string* ***Y-m-d***, un *date-time string*, una instancia de ***DateTime*** o una instancia de ***Carbon***.

### *Casting* de atributos

Podemos tener más control sobre el mapeo entre tipos de la base de datos y tipos *PHP* mediante este mecanismo. En el *array* ***\$casts*** podemos definir el tipo al que se mapeará un campo concreto de la tabla. Las claves del *array* son los nombres de los campos y los valores los tipos. Existen estos tipos predeterminados: ***integer***, ***real***, ***float***, ***double***, ***decimal:N***, ***string***, ***boolean***, ***object***, ***array***, ***collection***, ***date***, ***datetime*** y ***timestamp***.

En el caso de ***decimal*** se debe indicar el número de dígitos (p.e. ***decimal:5***).

```php
protected $casts = [
        'is_admin' => 'boolean',
        'nombre' => 'string',
        'fecha' => 'date'
    ];
```

A parte de los *casts* por defecto, se puede definir un *cast* a medida. Para ello debemos implementar una clase que implemente la interfaz ***Illuminate\Contracts\Database\Eloquent\CastsAttributes***. La clase debe definir las conversiones mediante el método `get()` (de base de datos a objeto *PHP*) y `set()` de objeto a base de datos. Luego, a la hora de definir el mapeo de un campo en el *array* ***\$casts***, se indicará como valor el nombre *fully qualified* de la clase (o mediante `::class`).
