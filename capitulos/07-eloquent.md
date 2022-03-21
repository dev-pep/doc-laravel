# Modelos: *Eloquent ORM*

El mecanimsmo *Eloquent ORM* (*object-relational mapping*, técnica de conversión entre tipos incompatibles, entre los tipos de una base de datos y los tipos en un lenguaje de programación orientado a objetos) proporciona los **modelos** de *Laravel*.

Los modelos se almacenan directamente en ***app***. Hay una forma fácil de crear un modelo, mediante `artisan`:

```
php artisan make:model Coche
```

Si además queremos que proceda a la creación de la migración correspondiente, no es necesario que creemos la migración por separado. Bastará con escribir:

```
php artisan make:model Coche --migration
```

Existen algunas **asunciones importantes** que hace *Eloquent*:

- El nombre de la tabla será el mismo que la clase modelo, pero **en plural y empezando en minúscula**. En este caso, el modelo ***Coche*** guardará sus registros en la tabla ***coches***. Si no queremos que se siga ese convenio, hay que definir el nombre de la tabla mediante la propiedad (*protected*) ***$table*** del modelo, que almacenará el nombre de la tabla.
- Cada tabla tiene un campo con nombre ***id***, que es la clave primaria. Para cambiarlo, definir una propiedad protegida en la clase del modelo llamada ***\$primaryKey***, con el nombre del campo con la clave primaria. También se asume que la clave primaria es un entero que se autoincrementa. Si queremos una clave primaria no autoincremental o no numérica, hay que definir la propiedad pública ***$incrementing*** a ***false***. Si el tipo no es numérico, además habría que definir la propiedad protegida ***$keyType*** a ***'string'***.
- Cada tabla tendrá las columnas ***created_at*** y ***updated_at***, que actualizará *Eloquent* automáticamente. Si deseamos que no lo haga, se incluirá la propiedad pública ***$timestamps*** con valor ***false***. Si queremos cambiar los nombres de los campos donde se guardarán los *timestamps*, se deben definir en la clase las constantes *CREATED_AT* y/o *UPDATED_AT*, conteniendo *strings* con los nombres deseados.

Si queremos dar un valor por defecto a alguno de los campos, definiremos la variable protegida ***$attributes***. Se le dará como valor un *array* con los nombres de los atributos a los que queramos dar valor por defecto, junto con sus valores:

```php
protected $attributes = [ 'delayed' => false, 'city' => 'Sabadell' ];
```

## Acceso a la base de datos

Una vez tenemos nuestro modelo, y su tabla correspondiente en la base de datos, podemos leer el contenido entero de la tabla:

```php
$coches = App\Coche::all();
```

El método `all()` retorna una *collection* (*array-like*) de modelos *Eloquent*. Estas *collections* aceptan los métodos definidos para *query builders*. Al ir encadenando métodos (tanto si son equivalentes a los de los *query builders* como si no) los resultados obtenidos son de tipo *collection Eloquent* también, con lo que al final no hay que llamar a `get()`. En cambio, si el primer método llamado es el correspondiente a un *query builder*, no retornará una *collection*, sino un *query builder Eloquent* (basado en un *query builder* convencional). En tal caso, podríamos ir refinando la *query*, y al final sí deberíamos usar `get()`:

```php
// Las dos sentencias son equivalentes:
$coches1 = App\Coche::all()
    ->where('id', '>', '5');
$coches2 = App\Coche::where('id', '>', '5')
    ->get();
```

Para pasar de colección a *query builder* y viceversa, disponemos de los métodos `toQuery()` (de las colecciones) y `fromQuery()` (de las *queries*).

Si en lugar de `where()` o `all()` indicamos como primer método `find()` con un *array* de claves primarias, recibiremos una **colección** de registros coincidentes (si los hay), no un *query builder*.

En el caso de una llamada a un método que retorna un solo registro (como `find()` con un solo valor, o `first()`), se puede obtener el valor actual del modelo en la base de datos mediante el método `refresh()`, por si ha habido cambios en el mismo por otro lado. Sin embargo, el modelo mantiene su valor antiguo, es decir, `refresh()` no cambia el valor del modelo; simplemente **retorna** el valor existente en base de datos.

De forma similar, el método `fresh()`, disponible en las *collections* de los modelos, retorna la consulta con datos frescos, sin afectar a la *collection* del modelo.

> Podemos definir tantos métodos como queramos en el modelo. Estos métodos pueden llamarse sobre objetos que contengan un solo registro. Los nombres de los campos de la tabla son atributos del objeto, que podrán ser accedidos a través de $this.

A la hora de acceder al valor de los campos, puede hacerse mediante el operador `->` y el nombre del campo, como un atributo del objeto. El objeto debe ser un modelo único (no una colección, ni un *query builder*). En ese sentido, sería válido el objeto retornado por `first()`, por ejemplo, o cada uno de los elementos de una colección *Eloquent*, por separado.

El objeto *Eloquent* dispone, internamente, de dos propiedades de tipo *array*: ***\$original*** y ***\$attributes***. Inicialmente, ambos contienen los valores por defecto (que hemos definido explícitamente), y, si procede, los valores en la base de datos al leer el modelo. A medida que se hacen cambios en el modelo con *PHP*, los cambios se van guardando en el *array* ***\$attributes***, pero ***\$original*** sigue igual. Cuando el modelo se guarda en base de datos, entonces ***\$original*** sí pasa a actualizarse según los valores de ***\$attributes***. Cuando se accede a un atributo del modelo, mediante `Coche->marca`, se retorna el valor correspondiente al contenido de ***\$attributes*** (o ***NULL*** si no existe el elemento).

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
