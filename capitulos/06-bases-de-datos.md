# Bases de datos

La conexión a la base de datos se configura en el archivo ***config/database.php***, que suele remitir a la información de configuración presente en ***.env*** (dirección, puerto, nombre base de datos, usuario, contraseña, etc.). Si la contraseña contiene espacios en blanco, se deben usar comillas dobles.

Una vez está configurada la base de datos adecuadamente, podemos acceder a través de la *facade* ***DB***. Esta, proporciona métodos para todos los tipos de *query* a la base de datos: `select()`, `insert()`, `update()`, `delete()` y `statement()`.

Para hacer un simple *select*, usaremos `select()`, que recibe dos argumentos: la consulta *SQL*, y un array de argumentos ligados a esta consulta (normalmente los valores del `where`):

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

El *query builder* es un mecanismo de acceso a los datos. Se trata de métodos que retornan una instancia de un *query builder* (***Illuminate\\Database\\Query\\Builder***), que es un objeto que representa una consulta *SQL*, asociada a una tabla específica. Se pueden ir añadiendo elementos a esa consulta, e irla refinando a base de métodos disponibles (que retornan un *query builder* refinado). Estos métodos se pueden ir encadenando uno tras otro.

Normalmente, los métodos encadenables van retornando a su vez un objeto *query builder*, hasta que recojamos los datos deseados en un último método que suele retornar otro tipo de datos. Un *query builder* no es un conjunto de resultados hasta que se le aplica un método, como por ejemplo `get()` que lo convierte en otra cosa, en este caso en una *collection* (***Illuminate\\Support\\Collection***), un tipo que funciona como un *array* de registros. Estos registros que componen la colección permiten también acceder a sus campos mediante la sintaxis de acceso a propiedades.

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

Para hacer un simpre select, método `select()` (retorna un *query builder*):

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

De forma similar las funciones de *PHP* `dd()` y `dump()`, disponemos de dos métodos de igual denominación y funcionamiento como terminales de la cadena (no hay que pasarles ningún argumento).

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

Esto ejecutará las nuevas migraciones que no se hayan aplicado todavía (archivos con *timestamp* posterior a la última migración (ejecución de `artisan migrate`).

Para revertir el última migración:

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

Ya hemos visto cómo definir campos a través del método `create()`. Existen otros métodos útiles en la *facade* ***Schema***: `hasTable()` comprueba si existe la tabla cuyo nombre le pasamos como argumento. `hasColumn()` comprueba si existe la columna; se le deben pasar dos nombres: tabla y columna. `rename()` cambia el nombre de una tabla (argumentos: nombre antiguo, nombre nuevo).

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

Si no lo hemos hecho en ese momento, se puede hacer posteriormente:

```php
$tabla->unique('email');
```

Si queremos crear un índice basado en varias columnas, le pasaremos un *array* con los nombres de las columnas:

```php
$tabla->unique(['email', 'nombre', 'fecha']);
```

Este método acepta un segundos parámetro que será el nombre del índice. Si no se especifica, se creará un nombre por defecto, basado en el nombre de la tabla, columnas, etc.

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

Naturalmente, antes de `constrained()` se pueden incluir otros métodos modificadores, como de costumbre.

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
- Cada tabla tiene un campo con nombre ***id***, que es la clave primaria. Para cambiarlo, definir una propiedad protegida en la clase del modelo llamada ***$primaryKey***, con el nombre del campo con la clave primaria. También se asume que la clave primaria es un entero que se autoincrementa. Si queremos una clave primaria no autoincremental o no numérica, hay que definir la propiedad pública ***$incrementing*** a ***false***. Si el tipo no es numérico, además habría que definir la propiedad protegida ***$keyType*** a ***'string'***.
- Cada tabla tendrá las columnas ***created_at*** y ***updated_at***, que actualizará *Eloquent* automáticamente. Si deseamos que no lo haga, se incluirá la propiedad pública ***$timestamps*** con valor ***false***.

Si queremos dar un valor por defecto a alguno de los campos, definiremos la variable protegida ***$attributes***. Se le dará como valor un *array* con los nombres de los atributos a los que queramos dar valor por defecto, junto con sus valores:

```php
protected $attributes = [ 'delayed' => false, 'city' => 'Sabadell' ];
```

### Acceso a la base de datos

Una vez tenemos nuestro modelo, y su tabla correspondiente en la base de datos, podemos leer el contenido entero de la tabla:

```php
$coches = App\Coche::all();
```

En lugar de `all()` se puede usar `where()` especificando una condición.

Es importante recalcar que algunos de estos métodos retornan una *collection* y otros un *query builder*. Por ejemplo, ***Coche::all()*** retorna una *collection*, mientras que ***Coche::where()*** retorna un *query builder*. Sin embargo, las colecciones disponen de muchos de los métodos que usamos para refinar *query builders*, con la única diferencia que los métodos de las colecciones retornarán a su vez una colección (haciendo posible, pues, el encadenado de métodos) que irán refinando la colección.

Por ejemplo:

```php
$coches1 = App\Coche::where('id', 10) -> get();
$coches2 = App\Coche::all() -> where('id', 10);
```

Las dos sentencias anteriores retornan el mismo valor y tipo. En la primera construimos una *query*, encadenamos uno o más métodos que van retornando *query builders*, y al final convertimos el último *query builder* retornado en una colección (mediante `get()`). En el segundo caso, ya tenemos una colección desde el principio, y vamos encadenando uno o más métodos que van retornando colecciones, hasta el último, que, al retornar también una colección, no se debe usar `get()`.

Si en lugar de `where()` o `all()` indicamos `find()` con un *array* de claves primarias, recibiremos una colección de registros coincidentes (si los hay).

En el caso de un *query builder*, se puede refrescar mediante `coches->refresh()`, por si ha habido cambios en la base de datos por otro lado. El método `fresh()`, en cambio, retorna la consulta con datos frescos, pero no cambia la *query* original. Las *collections* carecen de estos métodos.

### Insertar o modificar registros

Para añadir un registro, se crea una nueva instancia del modelo, se le dan los valores pertinentes, y se invoca el método `save()` del mismo.

```php
$coche = new Coche;
$coche->name =  $req->name;  // por ejemplo
$coche->save();
```

Si queremos modificar un registro:

```php
$coche = App\Coche::find(1);
$coche->name =  $req->name;  // por ejemplo
$coche->save();
```

Para inserciones y/o modificaciones, se pueden usar los métodos de *query builder* del modelo (`insert()`, `update()`, etc.). De hecho, se pueden usar estos métodos para lo que se quiera, aunque *Eloquent* añade más funcionalidad.
