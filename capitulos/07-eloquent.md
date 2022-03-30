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
protected $attributes = [ 'delayed' => false, 'city' => 'Sabadell' ];
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

Para pasar de colección a *builder* y viceversa, disponemos de los métodos `toQuery()` (de las colecciones) y `fromQuery()` (de las *queries*). Hay métodos que cambian de uno a otro automáticamente.

Si como primer método indicamos `find()` con un *array* de claves primarias, recibiremos, de forma similar a `all()`, una **colección** de registros coincidentes (si los hay), no un *builder*.

En el caso de los métodos que retornan un solo registro (como `find()` con un solo valor, no *array*, o `first()`), el tipo retornado es el del modelo (en este caso ***App\\Models\\Coche***). Estos métodos están disponibles tanto en colecciones como en *builders*, así como en el mismo modelo (por lo que pueden estar en primer lugar: `Coche::first()`).

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

<!-- TODO -->

## *Casting* de atributos

Podemos tener más control sobre el mapeo entre tipos de la base de datos y tipos *PHP* mediante este mecanismo. En la propiedad ***\$casts*** del modelo podemos definir mediante un *array* el tipo al que se mapeará cada campo concreto de la tabla. Las claves del *array* son los nombres de los campos y los valores son los tipos *PHP*. Estos son algunos de los tipos que existen por defecto: ***integer***, ***real***, ***float***, ***double***, ***decimal:N***, ***string***, ***boolean***, ***object***, ***array***, ***collection***, ***date***, ***datetime*** y ***timestamp***.

En el caso de ***decimal*** se debe indicar el número de dígitos (p.e. ***decimal:5***).

```php
protected $casts = [
        'es_electrico' => 'boolean',
        'marca' => 'string',
        'marca' => 'string',
        'km' => 'integer',
        'matriculacion' => 'date'
    ];
```

A parte de los *casts* por defecto, se puede definir un *cast* a medida. Para ello debemos implementar una clase que implemente la interfaz ***Illuminate\Contracts\Database\Eloquent\CastsAttributes***. La clase debe definir las conversiones mediante el método `get()` (de base de datos a objeto *PHP*) y `set()` de objeto a base de datos. Luego, a la hora de definir el mapeo de un campo en el *array* ***\$casts***, se indicará como valor el nombre *fully qualified* de la clase.

Tanto el método `get()` como el `set()` reciben como argumentos: una instancia del modelo, un *string* con la clave del campo, otro con el valor del mismo, y finalmente un *array* con atributos. Cada método retorna el valor del tipo adecuado (traducido).
