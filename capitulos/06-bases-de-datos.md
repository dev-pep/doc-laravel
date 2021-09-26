# Bases de datos

La conexión a la base de datos se configura en el archivo ***config/database.php***, que suele remitir a la información de configuración del entorno (o presente en ***.env***): dirección, puerto, nombre base de datos, usuario, contraseña, etc. Si la contraseña contiene espacios en blanco, se deben usar comillas dobles.

Una vez está configurada la base de datos adecuadamente, podemos acceder a través de la *facade* ***DB***. Esta, proporciona métodos para todos los tipos de *query* a la base de datos: `select()`, `insert()`, `update()`, `delete()` y `statement()`.

Para hacer un simple *select*, usaremos `select()`, que recibe dos argumentos: la consulta *SQL*, y un array de argumentos ligados a esta consulta (valores del filtro `where`, etc.):

```php
$datos = DB::select('select * from coches where marca = ?', ['seat']);
```

Los argumentos se van ligando a la consulta en orden de aparición de los ***'?'*** (es más seguro hacerlo así, para evitar ataques de *SQL injection*).

El método retorna un *array* con los registros resultantes. Para acceder a los campos se utiliza la sintaxis de acceso a propiedades:

```php
$m = $datos[2]->modelo;
```

En lugar de utilizar interrogantes, se pueden enlazar los argumentos con nombre:

```php
$datos = DB::select('select * from coches where marca = :mrc', ['mrc' => 'seat']);
```

Para insertar, se haría de forma similar:

```php
DB::insert('insert into users (id, name) values (?, ?)', [33501, 'Pepe']);
```

Un *update*:

```php
$affected = DB::update('update users set votes = 100 where name = ?', ['John']);
```

Este método retorna el número de registros afectados.

En cuanto a `delete()`, tendría la misma sintaxis, y también retorna el número de registros afectados.

Para ejecutar una sentencia cualquiera:

```php
DB::statement('drop table coches');
```

## *Query builders*

El *query builder* es un mecanismo de acceso a los datos. Se trata de métodos que retornan una instancia de un *query builder* (***Illuminate\\Database\\Query\\Builder***), que es un objeto que representa una consulta *SQL*, asociada a una tabla específica. Se pueden ir añadiendo elementos a esa consulta, e irla refinando a base de métodos disponibles (que retornan a su vez un *query builder* modificado). Estos métodos se pueden ir encadenando uno tras otro.

Normalmente, los métodos encadenables van retornando a su vez un objeto *query builder*, hasta que recojamos los datos deseados en un último método que suele retornar otro tipo de datos (típicamente un tipo *array-like*). Un *query builder* no es un conjunto de resultados hasta que se le aplica un método, como por ejemplo `get()` que lo convierte en otra cosa, en este caso en una *collection* (***Illuminate\\Support\\Collection***), un tipo que funciona como un *array* de registros. Estos registros que componen la colección permiten también acceder a sus campos mediante la sintaxis de acceso a propiedades.

El método inicial de creación de un *query builder* es `DB::table()`, que es un *select* de la tabla entera.

```php
$user = DB::table('users')->where('name', 'John')->first();
```

Este ejemplo toma la tabla ***users***, le aplica un *where* (`name=John`) mediante el método `where()`, y luego retorna el primero de los elementos. El método `first()` retorna el registro, no una colección ni un *array*. Este objeto retornado permite acceder a sus campos mediante sintaxis de acceso a propiedades.

`value()` retorna únicamente un valor: el del campo cuyo nombre le pasamos por parámtero, en el primer registro de la *query*.

`find()` retorna un simple objeto con el registro en que su clave primaria (campo ***id***) tiene el valor que pasamos como argumento.

`pluck()` retorna todos los valores que tiene la columna (campo) cuyo nombre le pasamos como argumento. El tipo de retorno es una *collection*.

Métodos agregación: `count()` (número de registros), `max('campo')`, `min('campo')`, `avg('campo')`, `sum('campo')`. Retornan un valor numérico.

Para comprobar si existen registros: `exists()` y `doesntExist()`. Retornan un booleano.

Para seleccionar campos, método `select()` (retorna un *query builder*):

```php
$users = DB::table('users')->select('name', 'email as user_email')->get();
```

El método `distinct()` retorna un *query builder* en el que se han eliminado los registros duplicados. El método `addSelect()` añade campos al *query builder*.

En un método como `select()`, `where()`, etc. se puede pasar como argumentos una *raw expression*. En este caso se usará el método `raw()`:

```php
$users = DB::table('users')
                 ->select(DB::raw('count(*) as user_count, status'))
                 ->where('status', '<>', 1)
                 ->groupBy('status')
                 ->get();
```

Dependiendo de a qué método deseemos pasar una expresión *raw*, podemos utilizar una sintaxis más breve, mediante los métodos `selectRaw()`, `whereRaw()`, `orWhereRaw()`, `groupByRaw`, etc.

### *Joins* y uniones

Para hacer un *inner join* con otra tabla, se usa el método `join()`. El primer argumeto es la tabla con la que hacer el *join*, y los siguientes especifican las restricciones de columna del mismo (se pueden encadenar varios *joins*):

```php
$users = DB::table('users')
            ->join('contacts', 'users.id', '=', 'contacts.user_id')
            ->join('orders', 'users.id', '=', 'orders.user_id')
            ->select('users.*', 'contacts.phone', 'orders.price')
            ->get();
```

De foma similar, podemos hacer un *left join* (`leftJoin()`) o un *right join* (`rightJoin()`). También se puede hacer un *cross join* (`crossJoin()`).

Con `union()` se pueden unir dos *query builders* (se le debe pasar como argumento el *query builder* a unir).

### *Where*

El método `where()` retorna un *query builder*. Espera como primer argumento el campo a comparar, como segundo, el operador de comparación, y como tercero el segundo operando de la comparación. Si queremos comparar por igualdad, se puede obviar el operador. Los operadores pueden ser ***=***, ***\<>***, ***>=***, ***like***, etc. Se puede pasar un *array* de condiciones:

```php
$users = DB::table('users')->where([
    ['status', '=', '1'],
    ['name', 'like', 'T%'],
])->get();
```

Se pueden encadenar varios `where()` (se aplica *and*), `orWhere()` (se aplica *or*), `whereIn()`, `whereNotIn()`, etc. Existen otros métodos como `whereDate()`, `whereMonth()`, `whereColumn()`, `orWhereColumn()`, etc.

### Agrupación, ordenación, limitación

Todos estos métodos son encadenables, esto es, retornan un *query builder*.

`orderBy()` toma como primer argumento la columna por la que ordenar, mientras que el segundo argumento puede ser ***'asc'*** o ***'desc'***. Si se quiere ordenar por varias columnas, se encadenan varios `orderBy()`.

Con `latest()` y `oldest()` se ordenan por fecha (por defecto según el campo ***created_at***). `inRandomOrder()` ordena aleatoriamente. `reorder()` elimina los órdenes anteriores.

Con `groupBy()` se agrupa por el campo (o campos) que se le pasa como argumento(s). Se puede combinar con `having()`.

Para limitar el número de registros resultante, tenemos el método `take()`, al que se le pasa el número de registros deseado. También está el método `skip()`, que se salta los primeros registros (el argumento especifica cuántos se saltarán).

> Al parecer, si usamos `skip()` debemos usar posteriormente `take()`.

### Inserciones

Para insertar, debemos pasar un registro (en un *array* de pares campo, valor) o varios (en un *array* de registros) a través del método `insert()`.

```php
DB::table('users')->insert([
    ['email' => 'taylor@example.com', 'votes' => 0],
    ['email' => 'dayle@example.com', 'votes' => 0],
]);
```

Si usamos `insertOrIgnore()` se ignorarán los registros duplicados. Si la tabla tiene una clave primaria autoincremental, asignara valor a esa clave si insertamos mediante `insertGetId()`, que retorna ese *id*.

### Cambios

De forma similar a las inserciones, un cambio se realiza mediante un *array* de pares campo, valor pasado al método `update()`; se puede restringir el cambio a los registros que especifiquemos mediante *wheres*.

```php
$affected = DB::table('users')
              ->where('id', 1)
              ->update(['votes' => 1]);
```

El método `update()` retorna el número de registros afectados.

El método `updateOrInsert()` toma dos argumentos: un *array* con una serie de condiciones, y otro con el registro a añadir a la base de datos. Si existen registros en la tabla que cumplen las condiciones, serán actualizados con el registro del segundo argumento. De lo contrario, ese registro será añadido a la tabla.

Para incrementar o decrementar un campo numérico, tenemos `increment()` y `decrement()`. Si se le pasa el nombre del campo, actuará en él. Si además le damos un número, ese cambio no será de 1 sino de lo que indiquemos.

### Borrado

Cuando la cadena de métodos termina en `delete()`, se borran los registros seleccionados. `truncate()` está pensado para eliminar todos los registros de la tabla y resetear el contador de *id* a 0.

### Depuración

De forma similar los *helpers* `dd()` y `dump()`, disponemos de dos métodos de igual denominación y funcionamiento como componentes de la cadena (no hay que pasarles ningún argumento).

## Migraciones

La base de datos puede administrarse desde una herramienta externa (como *phpMyAdmin* o similar), pero *Laravel* proporciona, como alternativa, un potente mecanismo para el mantenimiento a través de `artisan`. Este mecanismo permite trasladar fácilmente la estructura de la base de datos a otros desarrolladores y a personal de sistemas, de tal modo que el mismo `artisan` se encarga de hacer las modificaciones pertienentes en la estructura de la base de datos. Este mecanismo se realiza a través de las llamadas *migraciones*.

Una migración especifica la estructura de una tabla, y puede indicar cómo se crea esta, o una modificación de esta. Por ejemplo, para crear una migración que especifique cómo se debe producir la creación de una tabla ***coches***:

```
php artisan make:migration crea_tabla_coches --create=coches
```

En este caso, se creará una migración en el directorio ***database/migrations***. El nombre del archivo será ***\<timestamp>\_crea_tabla_coches.php***, donde ***\<timestamp>*** es una serie de caracteres que indican el momento de creación de la migración. Esto es necesario, puesto que si hay varios archivos de migración referidos a la misma tabla, se deberán aplicar en orden de creación. Con `--create` se indica que es una migración de creación y se proporciona el nombre de la tabla.

El archivo de migración del ejemplo contiene la clase (creada automáticamente) ***CreaTablaCoches***. En ella debe haber dos métodos: `up()`, que especifica el cambio o creación, y `down()`, opcional, que especifica cómo deshacer tal cambio (en caso de *rollback*).

La creación de una tabla se realiza mediante el método `create()` de la *facade* ***Schema***, que recibe como primer argumento el nombre de la tabla, y como segundo una *closure* que recibe un objeto ***Blueprint*** que servirá para definir la tabla nueva. Un ejemplo de migración para la creación de una tabla:

```php
class CreaTablaCoches extends Migration
{
    public function up()
    {
        Schema::create('coches', function (Blueprint $table) {
            $table->id();
            $table->char('marca', 50);
            $table->char('modelo', 40);
            $table->date('fecha_fabric')->nullable();
        });
    }

    public function down()
    {
        Schema::dropIfExists('coches');
    }
}
```

Para destruir una tabla, se usa también ***Schema***, y sus métodos `drop()` o `dropIfExists()`. Para renombrarla, `rename()`.

Si la migración no crea una tabla nueva, sino que modifica una existente, se usa `--table` en lugar de `--create`:

```
php artisan make:migration modifica1_tabla_coches --table=coches
```

En este caso, en lugar del método `create()`, utilizará el método `table()` para modificar la tabla.

Una vez creadas las migraciones que queramos, se ejecutarán así:

```
php artisan migrate
```

Esto ejecutará las nuevas migraciones que no se hayan aplicado todavía (archivos con *timestamp* posterior a la última migración, ejecutada con `artisan migrate`).

Para revertir la última migración:

```
php artisan migrate:rollback
```

Para revertir varios pasos a la vez:

```
php artisan migrate:rollback --step=5
```
Para revertir **todas** las migraciones del proyecto:

```
php artisan migrate:reset
```

### Definición de tablas

Ya hemos visto cómo definir campos a través del método `create()`. Existen otros métodos útiles en la *facade* ***Schema***: `hasTable()` comprueba si existe la tabla cuyo nombre le pasamos como argumento. `hasColumn()` comprueba si existe el campo; se le deben pasar dos nombres: tabla y columna. `rename()` cambia el nombre de una tabla (argumentos: nombre antiguo, nombre nuevo).

El objeto ***Blueprint*** dispone de numerosos métodos para crear campos (columnas). Los tipos de datos descritos son tipos *MySQL*. En general, estos métodos toman un argumento con el nombre del campo. Veamos algunos de ellos:

Para crear **claves primarias autoincrementales**: `bigIncrements()` es un ***UNSIGNED BIGINT***. `id()` equivale a `bigIncrements('id')`. También `increments()` (***UNSIGNED INTEGER***), `mediumIncrements()` (***UNSIGNED MEDIUMINT***), `smallIncrements()` (***UNSIGNED SMALLINT***), `tinyIncrements()` (***UNSIGNED TINYINT***).

Métodos para la creación de tipos numéricos: `bigInteger()` (equivale a ***BIGINT***), `boolean()` (***BOOLEAN***), `integer()` (***INTEGER***), `mediumInteger()` (***MEDIUMINT***), `smallInteger()` (***SMALLINT***), `tinyInteger()` (***TINYINT***), `decimal()` (***DECIMAL***, con tamaño total, y posiciones decimales), `double()` y `float()` (***DOUBLE*** y ***FLOAT***, con tamaño total, y posiciones decimales), `unsignedBigInteger()` (***UNSIGNED BIGINT***), `unsignedDecimal()` (***UNSIGNED DECIMAL***, con tamaño total, y posiciones decimales), `unsignedInteger()` (***UNSIGNED INTEGER***), `unsignedMediumInteger()` (***UNSIGNED MEDIUMINT***), `unsignedSmallInteger()` (***UNSIGNED SMALLINT***), `unsignedTinyInteger()` (***UNSIGNED TINYINT***).

El número de bits de estos enteros es el siguiente: ***TINYINT*** 8, ***SMALLINT*** 16, ***MEDIUMINT*** 24, ***INTEGER*** 32, y ***BIGINT*** 64.

Para tipos *string*: `binary()` (***BLOB***, *binary large object*), `char()` (***CHAR***, con tamaño en segundo argumento), `longtext()` (***LONGTEXT***), `mediumText()` (***MEDIUMTEXT***), `string()` (***VARCHAR***, con tamaño, por defecto 255), `text()` (***TEXT***), `enum()` (***ENUM***, con *array* con los valores posibles, de los que el campo solo puede tener un valor), `set()` (***SET***, con *array* con los valores posibles; a diferencia de un ***ENUM***, el valor del campo puede tener 0 o más elementos).

Para tipos fecha/hora: `date()` (***DATE***), `dateTime()` (***DATETIME***, con precisión), `time()` (***TIME***). El tipo ***DATETIME*** acepta un número de dígitos decimales, hasta 6 (microsegundos); el valor por defecto es 0.

Para otros tipos: `json()` (***JSON***).

A los métodos anteriores se les puede encadenar métodos modificadores:

`after('columna')` colocará el campo justo después del campo especificado.

`autoIncrement()` convierte un campo entero en autoincremental.

`comment('comentario')` añade un comentario a una columna.

`default($valor)` especifica un valor por defecto para esa columna.

`first()` coloca la columna en primera posición.

`nullable($value = true)` establece la columna como *nullable* (puede contener ***null***).

`unsigned()` establece una columna de tipo entero como *unsigned*.

### Índices

Se puede crear un índice sobre una columna, encadenándole simplemente `unique()` en la definición de la columna.

```php
$tabla->string('email')->unique();
```

Se puede hacer posteriormente a la creación del campo:

```php
$tabla->unique('email');
```

Si queremos crear un índice basado en varias columnas, le pasaremos un *array* con los nombres de las columnas:

```php
$tabla->unique(['email', 'nombre', 'fecha']);
```

Este método acepta un segundo parámetro que será el nombre del índice. Si no se especifica, se creará un nombre por defecto, basado en el nombre de la tabla, columnas, etc.

En lugar de `unique()`, se puede usar `index()` si queremos que se repitan valores en la columna. Otra alternativa es `primary()`, que lo que crea es una clave primaria.

Para **cambiar el nombre** de un índice, se usará el método `renameIndex()` al que se pasa el nombre antiguo y el nuevo.

Para **eliminar un índice**, dependiendo del tipo que sea, usaremos `dropPrimary()`, `dropUnique()` o `dropIndex()`, a los que pasaremos el nombre del índice.

### Restricciones de llave externa

Las *Foreign Key Constraints* son un mecanismo para ayudar a mantener la integridad referencial.

Supongamos que añadimos a una tabla existente ***posts*** una columna ***user_id***, que referencia a una clave primaria ***id*** en una tabla ***users***. Para añadir esta columna y crear esa restricción de clave externa:

```php
Schema::table('posts', function (Blueprint $table) {
    $table->unsignedBigInteger('user_id');  // crea la columna
    $table->foreign('user_id')->references('id')->on('users');  // crea la restricción
});
```

El código completo de la función puede sustituirse por una forma más resumida:

```php
$table->foreignId('user_id')->constrained();
```

Por un lado, `foreignId()` es un alias de `unsignedBigInteger()`. Y por otro lado, es importante el nombre que le demos a la nueva columna, puesto que es lo que utilizará *Laravel* para expandir al código anterior.

Naturalmente, antes de `constrained()` se pueden incluir otros métodos modificadores del campo, como de costumbre (es importante indicarse antes).

Por otro lado, es posible definir cláusulas ***ON DELETE*** y ***ON UPDATE*** mediante los métodos `onDelete()` y `onUpdate()` respectivamente, que se colocarán **después** de `constrained()`. En el caso, por ejemplo, de ***MySQL***, tenemos las siguientes posibilidades:

- ***ON DELETE***:
    - ***RESTRICT*** y ***NO ACTION*** son sinónimos, y el **comportamiento por defecto**. Levantan error si se intenta eliminar una registro referenciado en otra tabla como llave externa.
    - ***SET NULL*** pone las referencias a un valor nulo.
    - ***CASCADE*** elimina también los registros que referencian al registro a eliminar.

- ***ON UPDATE***:
    - ***RESTRICT*** y ***NO ACTION*** son sinónimos, y el **comportamiento por defecto**. Levantan error si se intenta cambiar la clave primaria referenciada en otra tabla como llave externa.
    - ***SET NULL*** pone las referencias de otras tablas a un valor nulo.
    - ***CASCADE*** cambia las referencias en las otras tablas.

```php
$table->foreignId('user_id')
    -> constrained()
    -> onDelete('restrict')
    -> onUpdate('cascade');
```

A la hora de definir la restricción, la tabla a la que pertenece la clave a la que hacemos referencia debe existir ya en la base de datos.

Para eliminar una de estas restricciones, es decir, para eliminar una *foreign key*, hay que usar `dropForeign()`, al que le pasaremos el nombre de esta. La convención del nombre es como sigue: nombre de la tabla, nombre del campo y el sufijo ***foreign***, todo separado por guiones bajos (***\_***). En nuestro caso sería:

```php
$table->dropForeign('posts_user_id_foreign');
```

Esto no elimina la columna, sino simplemente la restricción.

Para habilitar y deshabilitar las restricciones de clave externa en nuestras migraciones:

```php
Schema::enableForeignKeyConstraints();
Schema::disableForeignKeyConstraints();
```

## Modelos: *Eloquent ORM*

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

### Acceso a la base de datos

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

### Insertar o modificar registros

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

### Eliminación de registros

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

### *Mutators*

Este mecanismo permite gestionar el acceso a los atributos de un modelo *Eloquent*.

#### *Accessors*

Un *accessor* es un método definido dentro de la clase del modelo que define el acceso a un atributo concreto. El formato del nombre de este método debe ser del tipo ***get + NombreDelCampo + Attribute***, y se refiere a una columna de la tabla cuyo nombre es del tipo ***nombre_del_campo***. Obsérvese la disposición de mayúsculas y guiones bajos. Por ejemplo, el método ***getCilindradaMotorAttribute()*** se referiría al atributo (campo) del modelo ***cilindrada_motor***. El método debe retornar el valor del campo.

Si el campo ya existe en la tabla, este método permite manipular su valor, recibiendo el valor original en la tabla como primer parámetro:

```php
public function getCilindradaMotorAttribute($valor)
{
    return $valor . ' c.c.';
}
```

En cambio si no existe tal campo en la tabla, no recibe ningún valor, y permite retornar algún tipo de campo calculado, normalmente a partir del valor de otros campos del registro. Se puede acceder a ellos a través de `$this->atributo`.

#### *Mutators*

Un *mutator* tiene el mismo formato de nombre que un *accessor*, pero en lugar de retornar un valor, establece un valor de atributo. La forma correcta de hacerlo, es dando valor al elemento correspondiente de la propiedad ***\$attributes***:

```php
public function getMarcaAttribute($valor)
{
    $this->attributes['marca'] = strtolower($valor);
}
```

En este caso, suponiendo que ***\$registro*** contenga un modelo de ***Coche***, al hacer ```$registro->marca = 'BMW'```, el valor del atributo ***marca*** será ***bmw***, ya que el valor pasará a través del *mutator*, que lo pasa a minúsculas.

#### *Mutators* de fechas

Por defecto, *Eloquent* traduce los campos ***created_at*** y ***updated_at*** a tipo ***Carbon***, que es una extensión del ***DateTime*** de *PHP*. Si tenemos otros campos de fecha que deseamos que sean mapeados a tipo ***Carbon***, debemos añadir sus nombres al array ***\$dates***:

```php
protected $dates = ['fecha_inicio', 'fecha_fin'];
```

Cuando es así, podemos dar valor a un campo fecha del modelo usando un *timestamp UNIX*, un *date string* ***Y-m-d***, un *date-time string*, una instancia de ***DateTime*** o una instancia de ***Carbon***.

#### *Casting* de atributos

Podemos tener más control sobre el mapeo entre tipos de la base de datos y tipos *PHP* mediante este mecanismo. En el *array* ***\$casts*** podemos definir el tipo al que se mapeará un campo concreto de la tabla. Las claves del *array* son los nombres de los campos y los valores los tipos. Existen estos tipos predeterminados: ***integer***, ***real***, ***float***, ***double***, ***decimal:<digitos>***, ***string***, ***boolean***, ***object***, ***array***, ***collection***, ***date***, ***datetime*** y ***timestamp***.

```php
protected $casts = [
        'is_admin' => 'boolean',
        'nombre' => 'string',
        'fecha' => 'date'
    ];
```

A parte de los *casts* por defecto, se puede definir un *cast* a medida. Para ello debemos implementar una clase que implemente la interfaz ***Illuminate\Contracts\Database\Eloquent\CastsAttributes***. La clase debe definir las conversiones mediante el método `get()` (de base de datos a objeto *PHP*) y `set()` de objeto a base de datos. Luego, a la hora de definir el mapeo de un campo en el *array* ***\$cast***, se indicará como valor el nombre *fully qualified* de la clase (o mediante `::class`).
