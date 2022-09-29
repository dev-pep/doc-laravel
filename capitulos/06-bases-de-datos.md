# Bases de datos

La conexión a la base de datos se configura en el archivo ***config/database.php***, que suele leer la información de configuración de variables de entorno o del archivo ***.env***. En primer lugar se debe indicar la conexión a utilizar por defecto, con su *driver* correspondiente (***mysql***, ***sqlite***, ***pgsql***, ***sqlsrv***), y sus parámetros específicos: dirección, puerto, nombre base de datos, usuario, contraseña, etc. Si la contraseña contiene espacios en blanco, se deben usar comillas dobles.

> Es importante que la extensión *PHP* relativa al *driver* deseado esté instalada.

## Acceso mediante *queries*

Una vez está configurada la base de datos adecuadamente, podemos acceder a ella a través de la *facade* ***DB***. Esta, proporciona métodos para todos los tipos de *query* a la base de datos: `select()`, `insert()`, `update()`, `delete()` y `statement()`.

Para hacer un simple *select*, usaremos `select()`, que recibe dos argumentos: la consulta *SQL*, y un *array* opcional de argumentos ligados a esta consulta (valores del filtro `where`, etc.):

```php
$datos = DB::select('SELECT * FROM coches WHERE marca = ?',
                    ['seat']);
```

Los argumentos se van ligando a la consulta en orden de aparición de los ***'?'*** (es más seguro hacerlo así, para evitar ataques de *SQL injection*).

El método retorna un *array* numérico cuyos elementos son los registros resultantes. Para acceder a los campos de estos registros, se utiliza la sintaxis de acceso a propiedades:

```php
$m = $datos[2]->modelo;
```

En lugar de utilizar interrogantes, se pueden enlazar los argumentos con nombre:

```php
$datos = DB::select(
    'SELECT * FROM coches WHERE marca = :mrc',
    ['mrc' => 'seat']
);
```

Para insertar, se haría de forma similar:

```php
DB::insert('INSERT INTO users (id, name) VALUES (?, ?)',
           [33501, 'Pepe']);
```

Si la inserción tiene éxito, el método retorna *true*.

Un *update*:

```php
$affected = DB::update(
    'UPDATE users SET votes = 100 WHERE name = ?',
    ['John']
);
```

Este método retorna el número de registros afectados.

En cuanto a `delete()`, tendría la misma sintaxis, y también retorna el número de registros afectados.

Para ejecutar una sentencia *SQL* cualquiera:

```php
DB::statement('DROP TABLE coches');
```

### Selección de conexión

Si no deseamos que las *queries* se realicen con la conexión por defecto, podemos especificar el nombre de la conexión deseada en el momento de ejecutar la *query*:

```php
$datos =  DB::connection('sqlite')
    ->select('SELECT * FROM coches');
```

## Transacciones

Es posible agrupar varias *queries* en una sola transacción mediante el método `transaction()`, al que pasaremos una *closure* que contendrá todas las *queries*. Si dentro de la *closure* se llega a producir alguna excepción, se realizará el *rollback* automáticamente.

```php
DB::transaction(function () {
    DB::update('update users set votes = 1');

    DB::delete('delete from posts');
});
```

El método `transaction()` acepta como segundo argumento, opcional, un entero que indica el número máximo de reintentos de la transacción en caso de *deadlock*. Tras superar ese número, se levanta una excepción.

Otra forma de gestionar las transacciones, de forma más manual y con más control, es indicando que la transacción empieza mediante `DB::beginTransaction()`. A partir de este punto, se puede generar un *commit* mediante `DB::commit()`, o un *rollback* con `DB::rollBack()`.

Todos estos métodos de gestionar transacciones con la *facade* ***DB*** funcionan también con *query builders* y con modelos *Eloquent* (ver más adelante).

## Conexión al *CLI* de la base de datos

Es posible ejecutar el cliente de línea de comandos de la conexión de base de datos por defecto:

```
php artisan db
```

Si se quiere ejecutar el cliente de otra conexión, se debe especificar el nombre de esta:

```
php artisan db pgsql
```

## *Query builders*

El *query builder* es un mecanismo que permite construir *queries* a la base de datos mediante el encadenamiento de métodos. Las *queries* construidas así son inmunes a los ataques de inyección de *SQL*.

Cada uno de estos métodos retorna una instancia de un *query builder* (***Illuminate\\Database\\Query\\Builder***). Se puede ir refinando la *query* en la cadena de métodos. Al final de la cadena utilizaremos un método que transforme el *query builder* deseado en una *collection* (***Illuminate\\Support\\Collection***) de registros (método `get()`).

El método inicial para empezar el *query builder* es `DB::table()`. Empezaremos simplemente indicando la tabla sobre la que trabajar (se trata de un *select* de la tabla entera), indicada como argumento *string*. Este método retorna, lógicamente, una instancia del *query builder*.

El método final es frecuentemente `get()`, un método del objeto *query builder* que retorna una *collection* cuyos elementos son objetos genéricos correspondientes a los registros del *query builder*. Estos objetos tienen como propiedades los campos de la tabla.

```php
$tablaCoches = DB::table('coches')->get();
$marcaCoche2 = $tablaCoches[2]->marca;
```

> Una collection puede convertirse en *array* mediante el método `toArray()` de la collection.

Una forma de aplicar un filtro al query builder es mediante `where()` (se verá en detalla más adelante). Por otro lado, en lugar de convertir el *query builder* en una *collection* de objetos, se puede convertir en un solo objeto mediante el método `first()`. En este caso, el objeto es el primero de los registros de la *query*.

```php
$user = DB::table('coches')
    ->where('marca', 'Volvo')
    ->first();
```

Este ejemplo toma la tabla ***coches***, le aplica un *where* (`marca=Volvo`) mediante el método `where()`, y luego retorna el primero de los elementos. El método `first()` retorna el registro, no una colección. Este objeto retornado permite acceder a sus campos mediante sintaxis de acceso a propiedades.

`value()` retorna únicamente un valor simple: el del campo cuyo nombre le pasamos por parámetro (*string*), en el primer registro de la *query*. El resultado retornado es un *string*.

`find()` retorna un simple objeto con el registro tal que su clave primaria (campo con nombre ***id***) tiene el valor que pasamos como argumento. Si no encuentra dicho registro, retorna ***null***.

`pluck()` retorna todos los valores que tiene la columna (campo) cuyo nombre le pasamos como argumento. El tipo de retorno es una *collection*. Si le pasamos un segundo argumento con en nombre de otro campo, los elementos de la *collection* retornada tendrán como clave el valor de ese segundo campo.

### Fragmentos (*chunks*)

Si hay que trabajar sobre miles de registros, en lugar de leer todos los registros de golpe y trabajar sobre ellos, se pueden ir leyendo por fragmentos de un número determinado de registros. Para ello, el método `chunk()` lo hace automáticamente. El método espera el tamaño máximo del fragmento, y una *closure* con el código a aplicar a esos registros. La *closure* recibe una *collection* con los registros del *chunk* como objetos estándar. Para garantizar que el orden de los registros es el mismo cada vez que se lee un *chunk*, es obligatorio especificar una cláusula *order by* de *SQL* para utilizar este método. Más adelante se verá el método `orderBy()`:

```php
DB::table('coches')->orderBy('id')
    ->chunk(100, function($registros) {
    foreach($registro as $registros)
        { /* ... */ }
});
```

El mecanismo del ejemplo es el siguiente: se leen los primeros 100 registros. Se procesan con el código de la *closure*. Si esta retorna ***false***, se termina la ejecución. En caso contrario, se lee el siguiente *chunk*, hasta que se han procesado todos los registros de la tabla (el último *chunk* puede ser inferior a 100 elementos).

El problema puede aparecer cuando a medida que se van procesando los archivos se insertan o eliminan registros. Esto puede hacer que los fragmentos obtenidos no sean correctos. Para evitarlo, en lugar de fragmentar según el orden, se puede hacer por *id*. Es decir, no se trata de ordenar por *id* y tomar los N primeros, los N siguientes, etc. Se trata de tomar los registros según el valor de dicho *id*. En este caso, añadir y borrar registros es seguro, aunque cambiar el valor del campo *id* de los registros puede conllevar problemas a su vez.

```php
DB::table('coches')->chunkById(100, function ($registros) {
    foreach($registro as $registros)
        { /* ... */ }
});
```

En este caso no es necesario el *order by*.

### Agregación

Métodos agregación: `count()` (número de registros), `max()`, `min()`, `avg()`, `sum()`. A excepción del primero, toman un argumento con el nombre del campo deseado. Retornan un valor numérico.

Para comprobar si existen registros: `exists()` y `doesntExist()`. Retornan un booleano.

### Sentencias *select*

Para seleccionar campos, se puede refinar el *query builder* con el método `select()`:

```php
$users = DB::table('users')
    ->select('name', 'email as user_email')
    ->get();
```

El método `distinct()` retorna un *query builder* en el que se han eliminado los registros duplicados. El método `addSelect()` añade campos al *query builder*.

En un método como `select()`, `where()`, etc. se puede pasar como argumentos una expresión *SQL* literal (*raw expression*). En este caso se usará el método `raw()`:

```php
$users = DB::table('users')
    ->select(DB::raw('count(*) as user_count, status'))
    ->where('status', '<>', 1)
    ->groupBy('status')
    ->get();
```

Dependiendo de a qué método deseemos pasar una expresión literal, podemos utilizar una sintaxis más compacta, mediante los métodos `selectRaw()`, `whereRaw()`, `orWhereRaw()`, `havingRaw()`, `orHavingRaw()`, `orderByRaw()` o `groupByRaw()`.

### *Joins* y uniones

Para hacer un *inner join* con otra tabla, se usa el método `join()`. El primer argumento es la tabla con la que hacer el *join*, y los siguientes especifican las restricciones de columna del mismo (se pueden encadenar varios *joins*):

```php
$users = DB::table('users')
    ->join('contacts', 'users.id', '=', 'contacts.user_id')
    ->join('orders', 'users.id', '=', 'orders.user_id')
    ->select('users.*', 'contacts.phone', 'orders.price')
    ->get();
```

En el caso de los *joins* podríamos tener un problema cuando existiesen campos con nombres idénticos en las tablas unidas (se sobreescribirían). En este caso es posible dar un **alias a los campos** que deseemos en el método `select()`. Supongamos que en la tabla ***users*** del ejemplo anterior ya existe un campo ***phone***. En este caso, tendríamos un conflicto entre ***users.phone*** y ***contacts.phone***. Si no hacemos nada para evitarlo, solo uno de los dos campos aparecerá en los registros (con el nombre ***phone***). Podemos solucionarlo con un alias:

```php
// ...
    ->select('users.*', 'contacts.phone AS tel',
             'orders.price')
    // ...
```

También es posible dar **alias a los nombres de las tablas** que deseemos. En ese caso, se deberá usar el alias y no el nombre de la tabla:

```php
$users = DB::table('users AS u')
    ->join('contacts AS c', 'u.id', '=', 'c.user_id')
    ->join('orders AS o', 'u.id', '=', 'o.user_id')
    ->select('u.*', 'c.phone AS tel', 'o.price')
    ->get();
```

De forma similar, podemos hacer un *left join* (`leftJoin()`) o un *right join* (`rightJoin()`).

También se puede hacer un *cross join* (`crossJoin()`). En este caso solo se especifica como parámetro la tabla con la que hacer el producto cartesiano.

Con `union()` se pueden unir dos *query builders* (se le debe pasar como argumento el *query builder* a unir).

### *Where*

El método `where()` retorna un *query builder* filtrado según la condición indicada. Espera como primer argumento el campo a comparar, como segundo el operador de comparación, y como tercero el segundo operando de la comparación. Si queremos comparar por igualdad, se puede obviar el operador. Los operadores pueden ser ***=***, ***\<>***, ***>=***, ***like***, etc. Se puede, alternativamente, pasar un *array* de condiciones:

```php
$users = DB::table('users')->where([
    ['status', '=', '1'],
    ['name', 'like', 'T%'],
])->get();
```

Se pueden encadenar varios `where()` (se aplica *and*), `orWhere()` (se aplica *or*), `whereNot()`, `orWhereNot()`, `whereBetween()`, `orWhereBetween()`, `whereNotBetween()`, `orWhereNotBetween()`, `whereIn()`, `whereNotIn()`, `orWhereIn()`, `orWhereNotIn()`, etc.

Para agrupar cláusulas *where* adecuadamente teniendo en cuenta las *and* y *or*, puede ser necesario usar *closures*:

```php
$users = DB::table('users')
    ->where('name', '=', 'John')
    ->where(function ($query) {
        $query->where('votes', '>', 100)
            ->orWhere('title', '=', 'Admin');
        })
    ->get();
```

### Orden

El método `orderBy()` toma como primer argumento la columna por la que ordenar, mientras que el segundo argumento puede ser un *string* con ***asc*** o ***desc***. Si se quiere ordenar por varias columnas, se deben encadenan varios `orderBy()`.

Con `latest()` y `oldest()` se ordenan por fecha (por defecto según el campo ***created_at***). `inRandomOrder()` ordena aleatoriamente. `reorder()` elimina las ordenaciones anteriores.

### Agrupación

Con `groupBy()` se agrupa por el campo (o campos) que se le pasa como argumento(s). Se puede combinar con `having()` (y otros como `havingBetween()`).

### Límites y *offset*

Para limitar el número de registros resultante, tenemos el método `take()` (o `limit()`), al que se le pasa el número de registros deseado. También está el método `skip()` (u `offset()`), que se salta los primeros registros (el argumento especifica cuántos se saltarán).

Si usamos `skip()` (u `offset()`) debemos usar también `take()` (o `limit()`).

### Cláusulas condicionales

Es posible aplicar un método en la cadena solamente si una determinada condición es verdadera, a través del método `when()`.

```php
$coches = DB::table('coches')
    ->when($marca, function($query, $marca) {
        $query->where('marca', $marca);
    })
    ->get();
```

En este caso, al *query builder* se le aplicará la cláusula *where* cuando la variable ***\$marca*** exista y se evalúe a ***true***. La *closure* como primer argumento el *query builder* que estamos construyendo, y como segundo, el primer argumento a `when()`.

### Inserciones

Para insertar, debemos pasar al *query builder* un registro (en un *array* de pares campo/valor) o varios (en un *array* de registros) a través del método `insert()`.

```php
DB::table('users')->insert([
    ['email' => 'taylor@example.com', 'votes' => 0],
    ['email' => 'dayle@example.com', 'votes' => 0],
]);
```

Si en su lugar usamos `insertOrIgnore()` se ignorarán los registros duplicados. Si la tabla tiene una clave primaria autoincremental, asignará valor a esa clave si insertamos mediante `insertGetId()`, que retorna además ese *id*.

### *Upserts*

Un *upsert* es un híbrido entre un *insert* y un *update*. A través del método `upsert()`, se especifican una serie de registros. Para cada registro, si este tiene una coincidencia en la tabla, se actualizará adecuadamente. De lo contrario, el registro será insertado.

Para ello, el método `upsert()` recibe un primer argumento consistente en un *array* de registros. Cada registro es a su vez un *array* con pares clave/valor (campos con sus valores correspondientes). El segundo argumento es un *array* con los nombres de los campos que se usarán para comprobar la existencia del registro en la base de datos. El tercer *array* indica el campo a actualizar en caso de que el registro exista en la tabla.

```php
DB::table('flights')->upsert([
    [
        'departure' => 'Oakland',
        'destination' => 'San Diego',
        'price' => 99
    ],
    [
        'departure' => 'Chicago',
        'destination' => 'New York',
        'price' => 150
    ]
], ['departure', 'destination'], ['price']);
```

En este ejemplo, se intentan insertar estos dos registros. Pero si existe ya un registro cuyos campos ***departure*** y ***destination*** coinciden con el registro a insertar, simplemente se actualizará el valor de su columna ***price***.

> A excepción de SQL Server, será necesario que el segundo argumento de `upsert()` contenga por lo menos un campo único o una clave primaria.

### Actualizaciones

De forma similar a las inserciones, una actualización o cambio (*update*) se realiza mediante un *array* de pares campo/valor pasado al método `update()`; se puede restringir el cambio a los registros que especifiquemos mediante *wheres*.

```php
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['votes' => 1]);
```

El método `update()` retorna el número de registros afectados.

El método `updateOrInsert()` toma dos argumentos: un *array* con una serie valores de campos (uno o varios elementos clave/valor), y otro con los campos a actualizar (también uno o más campos con sus valores). Si existen registros en la tabla que coinciden (campos del primer argumento), recibirán la actualización de los campos indicados en el segundo argumento. De lo contrario, ese registro será añadido a la tabla, con los campos de ambos argumentos combinados.

Para incrementar o decrementar un campo numérico, tenemos `increment()` y `decrement()`. Debemos indicar el nombre del campo. Si además especificamos un número (siempre positivo), ese cambio no será de 1 sino de lo que indiquemos. Si además indicamos un *array* con pares campo/valor, ese campo será también actualizado.

```php
DB::table('coches')
    ->increment('km', 100, ['estado' => 'usado']);
```

### Borrado

Cuando la cadena de métodos termina en `delete()`, se borran los registros seleccionados. El método `truncate()` está pensado para eliminar **todos** los registros de la tabla y resetear el contador de *id* a 0; la cadena debería tener solo dos métodos: `table()` y `truncate()`.

### Bloqueos

Para que toda la transacción se ejecute con un *shared lock* (usado para lectura), se incluirá en la cadena el método `sharedLock()`. Este tipo de bloqueo no permite la escritura por parte de otro proceso hasta que hemos terminado de obtener los datos.

Por otro lado, si lo que deseamos es escribir en la tabla, necesitamos un bloqueo para escritura, lo cual se consigue mediante la inclusión de `lockForUpdate()`, que no permite ni la escritura ni la selección hasta finalizar la transacción.

### Depuración

Los métodos `dd()` y `dump()` proporcionan información sobre la consulta *SQL* asociada al *query builder* en ese momento. El primero termina la ejecución. Por otro lado, el método `toSql()` retorna un *string* con la consulta *SQL*.

## Migraciones

La base de datos puede administrarse desde una herramienta externa (como *phpMyAdmin* o similar), pero *Laravel* proporciona, como alternativa, un potente mecanismo para el mantenimiento a través de `artisan`. Este mecanismo permite trasladar fácilmente la estructura de la base de datos a otros desarrolladores y al personal de sistemas encargado de los despliegues, de tal modo que es `artisan` quien se encarga de crear y modificar la estructura de la base de datos adecuadamente. Este mecanismo se realiza a través de las llamadas *migraciones*.

Una migración especifica la estructura de una tabla, y puede indicar cómo se crea esta, o cómo debe modificarse. Por ejemplo, para crear una migración que especifique cómo se debe producir la creación de una tabla ***coches***:

```
php artisan make:migration create_coches_table
```

En este caso, se creará una migración en el directorio ***database/migrations***. El nombre del archivo será ***\<timestamp>\_create_coches_table.php***, donde ***\<timestamp>*** es una serie de caracteres que indican el momento de creación de la migración. Esto es necesario, puesto que si hay varios archivos de migración referidos a la misma tabla, se deberán aplicar en orden de creación.

*Laravel* guardará registro de las migraciones que se han aplicado ya, para no repetirlas. Ello lo hace mediante una tabla ***migrations*** en la misma base de datos.

*Laravel* intentará deducir el nombre de la tabla a partir del nombre de la migración, y si se trata de una tabla que hay que crear, o una que hay que modificar. En el ejemplo, la tabla será ***coches***. En caso de que no sea posible tal deducción, simplemente habrá que editar el archivo de migración.

Dicho archivo, en el ejemplo, contiene una clase anónima (`return new class extends Migration`). De este modo, no hay colisión de nombres en las migraciones. En esta clase debe haber dos métodos: `up()`, que especifica el cambio o creación, y `down()`, opcional, que especifica cómo deshacer tal cambio (en caso de *rollback*).

La creación de una tabla se realiza mediante el método `create()` de la *facade* ***Schema***, que recibe como primer argumento el nombre de la tabla, y como segundo una *closure* que a su vez recibe un objeto ***Illuminate\\Database\\Schema\\Blueprint***, el cual servirá para definir la tabla nueva. Si *Laravel* puede deducir a través del nombre que se trata de una migración de creación, se creará este código automáticamente:

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::create('coches', function (Blueprint $t) {
            $t->id();
            $t->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('coches');
    }
};
```

Para destruir una tabla, se usa también ***Schema***, y sus métodos `drop()` o `dropIfExists()`. Para renombrarla, `rename()`.

Si la migración está pensada para una conexión concreta de base de datos, hay que especificarlo en la propiedad ***\$connection*** de la migración:

```php
protected $connection = 'pgsql';
```

En las migraciones de modificación, en lugar del método `create()` se usa el método `table()` para modificar la tabla.

### Ejecución de las migraciones

Una vez creadas las migraciones que queramos, se ejecutarán así:

```
php artisan migrate
```

Esto ejecutará las nuevas migraciones que no se hayan aplicado todavía, y quedarán registradas como ejecutadas (en la tabla ***migrations***).

Para ver el estado de las migraciones, es decir, qué migraciones se han aplicado actualmente:

```
php artisan migrate:status
```

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

El comando de *artisan* `migrate:refresh` realiza *rollback* de todas las migraciones ejecutadas y posteriormente ejecuta el comando `migrate`, es decir, regenera la base de datos completamente.

En cambio, `migrate:fresh` lo que hace es eliminar todas las tablas existentes en la base de datos (fruto de migraciones o no), borrar el registro de migraciones realizadas, y ejecutar `migrate`.

### Creación de tablas

Ya hemos visto cómo definir campos a través del método `create()` de la *facade* ***Schema***. Existen otros métodos útiles esta *facade*: `hasTable()` comprueba si existe la tabla cuyo nombre le pasamos como argumento. `hasColumn()` comprueba si existe el campo; se le deben pasar dos nombres: tabla y columna. `rename()` cambia el nombre de una tabla (argumentos: nombre antiguo, nombre nuevo).

El método `table()` de ***Schema*** tiene el mismo uso que `create()`, pero en lugar de crear la tabla la modifica.

Si deseamos especificar una conexión de base de datos en lugar de usar la conexión por defecto, se debe usar el método `connection()`:

```php
Schema::connection('sqlite')
    ->create('tabla', function(Blueprint $table) {
        /*...*/
    });
```

En el caso específico de *MySQL*/*MariaDB*, se pueden indicar algunas características de la tabla a crear mediante determinadas propiedades del objeto ***Blueprint***:

- ***engine*** permite indicar el motor de base de datos (normalmente ***MyISAM*** o ***InnoDB***).
- ***charset*** es el juego de caracteres de la tabla.
- ***collation*** es la *collation* de la tabla, es decir, el modo utilizados para comparar y ordenar caracteres y *strings*.

De no indicarse estos valores, se usarán los valores por defecto que tenga *MySQL*/*MariaDB*.

```php
Schema::create('coches', function (Blueprint $table) {
    $table->engine = 'InnoDB';
    $table->charset = 'utf8mb4';
    $table->collation = 'utf8mb4_unicode_ci';
    // ...
});
```

El objeto ***Blueprint*** dispone de numerosos métodos para crear campos (columnas). Los tipos de datos descritos son tipos propios de la base de datos (normalmente *SQL*). En general, estos métodos toman un argumento con el nombre del campo; algunos admiten otros argumentos. Veamos algunos de ellos (asumiremos tipos *MySQL*):

Para crear **claves primarias autoincrementales**: `bigIncrements()` es un ***UNSIGNED BIGINT***. `id()` equivale a `bigIncrements('id')`, y no precisa argumento. También `increments()` (***UNSIGNED INTEGER***), `mediumIncrements()` (***UNSIGNED MEDIUMINT***), `smallIncrements()` (***UNSIGNED SMALLINT***), `tinyIncrements()` (***UNSIGNED TINYINT***).

Métodos para la creación de tipos numéricos: `bigInteger()` (equivale a ***BIGINT***), `boolean()` (***BOOLEAN***), `integer()` (***INTEGER***), `mediumInteger()` (***MEDIUMINT***), `smallInteger()` (***SMALLINT***), `tinyInteger()` (***TINYINT***), `decimal()` (***DECIMAL***, con tamaño total, y posiciones decimales), `double()` y `float()` (***DOUBLE*** y ***FLOAT***, con tamaño total, y posiciones decimales), `unsignedBigInteger()` (***UNSIGNED BIGINT***), `unsignedDecimal()` (***UNSIGNED DECIMAL***, con tamaño total, y posiciones decimales), `unsignedInteger()` (***UNSIGNED INTEGER***), `unsignedMediumInteger()` (***UNSIGNED MEDIUMINT***), `unsignedSmallInteger()` (***UNSIGNED SMALLINT***), `unsignedTinyInteger()` (***UNSIGNED TINYINT***).

El número de bits de estos enteros es el siguiente: ***TINYINT*** 8, ***SMALLINT*** 16, ***MEDIUMINT*** 24, ***INTEGER*** 32, y ***BIGINT*** 64.

Para tipos *string*: `binary()` (***BLOB***, *binary large object*), `char()` (***CHAR***, con tamaño en segundo argumento), `longtext()` (***LONGTEXT***), `mediumText()` (***MEDIUMTEXT***), `string()` (***VARCHAR***, con tamaño, por defecto 255), `text()` (***TEXT***), `enum()` (***ENUM***, con *array* con los valores posibles, de los que el campo solo puede tener un valor), `set()` (***SET***, con *array* con los valores posibles; a diferencia de un ***ENUM***, el valor del campo puede tener 0 o más elementos).

Para tipos fecha/hora: `date()` (***DATE***), `dateTime()` (***DATETIME***, con precisión), `time()` (***TIME***). El tipo ***DATETIME*** acepta un número de dígitos decimales, hasta 6 (microsegundos); el valor por defecto es 0.

Para otros tipos: `json()` (***JSON***).

El método `timestamps()` crea los campos ***created_at*** y ***updated_at***, que son utilizados automáticamente por *Eloquent* (ver más adelante).

A los métodos anteriores se les puede encadenar métodos modificadores:

`after('columna')` colocará el campo justo después del campo especificado.

`autoIncrement()` convierte un campo entero en autoincremental.

`comment('comentario')` añade un comentario a una columna.

`default($valor)` especifica un valor por defecto para esa columna.

`first()` coloca la columna en primera posición.

`nullable($value=true)` establece la columna como *nullable* (puede contener ***null***).

`unsigned()` establece una columna de tipo entero como *unsigned*.

### Modificación de columnas

Se usa el método `Schema::table()` para la modificación de tablas existente. Si se desea añadir nuevas columnas, se hace del mismo modo que en `Schema::create()`.

Si necesitamos modificar columnas ya existentes, es necesario instalar el paquete *doctrine/dbal*:

```
composer require doctrine/dbal
```

> Hay otras configuraciones a realizar si deseamos modificar campos creados con `timestamp()`.
>
> Si la base de datos es *SQL Server*, hay que instalar `doctrine/debal:^3.0`.

Si redefinimos una columna ya existente, hay que hacerlo como en el caso de la creación de columnas, pero se debe encadenar al final el método `change()`:

```php
Schema::table('coches', function (Blueprint $tabla) {
    $tabla->string('marca', 50)->nullable()->change();
});
```

Si se desea renombrar una columna, se usa el método `renameColumn()`, al que se debe indicar el nombre antiguo y el nuevo.

### Eliminación de columnas

En este caso también es necesario el paquete *doctrine/dbal*, pero solo cuando trabajemos con bases de datos *SQLite*.

Para eliminar una columna existente, se usa el método `dropColumn()` del objeto ***Blueprint***, al que se le pasa el nombre de la columna, o un *array* de nombres de columnas (en el caso de *SQLite* solo acepta la eliminación de una columna a la vez).

### Índices

Se puede crear un índice sobre una columna, encadenando simplemente `unique()` en la definición de dicha columna.

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

En lugar de `unique()`, se puede usar `index()` si queremos que se repitan valores en la columna. Otra alternativa es `primary()`, que lo que crea es una clave primaria. Todos estos métodos aceptan un segundo argumento opcional con el nombre del índice.

Para **cambiar el nombre** de un índice, se usará el método `renameIndex()` al que se pasa el nombre antiguo y el nuevo.

Para **eliminar un índice**, dependiendo del tipo que sea, usaremos `dropPrimary()`, `dropUnique()` o `dropIndex()`, a los que pasaremos el nombre del índice.

### Restricciones de llave externa

Las restricciones de llave externa (*foreign key constraints*) son un mecanismo para ayudar a mantener la integridad referencial.

Supongamos que añadimos a una tabla ***coches*** una columna ***color_id***, que referencia a una clave primaria ***id*** en una tabla ***colores***. Para añadir esta columna y crear esa restricción de clave externa:

```php
Schema::table('coches', function (Blueprint $tabla) {
    $tabla->unsignedBigInteger('color_id');  // crea columna
    $tabla->foreign('color_id')
        ->references('id')
        ->on('colors');  // crea la restricción
});
```

El código completo de la función puede sustituirse por una forma más resumida:

```php
$table->foreignId('color_id')->constrained();
```

Por un lado, `foreignId()` es un alias de `unsignedBigInteger()`. Y por otro lado, es importante el nombre que le demos a la nueva columna, puesto que es lo que utilizará *Laravel* para deducir y expandir el código anterior. En caso de que el nombre de la tabla no siga la convención de *Laravel*, debe indicarse el nombre de la misma al método `constrained()`. Supongamos que en lugar de ***colors*** la tabla se llama ***colores*** (lo cual no sigue el convenio):

```php
$table->foreignId('color_id')->constrained('colores');
```

Naturalmente, **antes** de `constrained()` se pueden incluir otros métodos modificadores del campo, como de costumbre.

Por otro lado, es posible definir cláusulas ***ON DELETE*** y ***ON UPDATE*** mediante los métodos `onDelete()` y `onUpdate()` respectivamente, que se colocarán **después** de `constrained()`. En el caso, por ejemplo, de ***MySQL***, tenemos las siguientes posibilidades:

- ***ON DELETE***:
    - ***RESTRICT*** y ***NO ACTION*** son sinónimos, y el **comportamiento por defecto**. Producen un error si se intenta eliminar un registro referenciado en otra tabla como llave externa.
    - ***SET NULL*** pone un valor nulo a las referencias al registro eliminado.
    - ***CASCADE*** elimina también los registros que referencian al registro eliminado.
- ***ON UPDATE***:
    - ***RESTRICT*** y ***NO ACTION*** son sinónimos, y el **comportamiento por defecto**. Producen error si se intenta cambiar la clave primaria referenciada en otra tabla como llave externa.
    - ***SET NULL*** pone un valor nulo a las referencias a la clave modificada.
    - ***CASCADE*** cambia el valor de las referencias para que coincidan con la clave modificada.

```php
$table->foreignId('color_id')
    ->constrained()
    ->onDelete('restrict')
    ->onUpdate('cascade');
```

A la hora de definir la restricción, la tabla a la que pertenece la clave a la que hacemos referencia debe existir ya en la base de datos.

Por conveniencia existen métodos como `cascadeOnUpdate()`, `restrictOnUpdate()`, `cascadeOnDelete()`, `restrictOnDelete()`, y `nullOnDelete()`.

Para eliminar una de estas restricciones y eliminar una *foreign key*, hay que usar `dropForeign()`, al que le pasaremos el nombre de esta. El convenio del nombre es como sigue: nombre de la tabla, nombre del campo y sufijo ***foreign***, todo separado por guiones bajos (***\_***). En nuestro caso sería:

```php
$table->dropForeign('coches_color_id_foreign');
```

Esto no elimina la columna, sino simplemente la clave externa (y la correspondiente restricción).

Para habilitar y deshabilitar las restricciones de clave externa en nuestras migraciones:

```php
Schema::enableForeignKeyConstraints();
Schema::disableForeignKeyConstraints();
```

> *SQLite* deshabilita las claves externas por defecto. Debemos asegurarnos de habilitarlas en ***config/database.php***.

## *Seeding*

El proceso para rellenar las tablas con datos de ejemplo (aleatorios en muchos casos) se denomina *seeding*.

Para ello, *Laravel* utiliza unas clases llamadas *seeders*. Para crear un *seeder*:

```
php artisan make:seeder CochesSeeder
```

Esto creará la clase ***Database\\Seeders\\CochesSeeder*** (en el archivo ***database/seeders/CochesSeeder.php***). Un seeder tiene un método `run()` que es ejecutado cuando se invoca al *seeder* (a través de *artisan*):

```
php artisan db:seed --class=CochesSeeder
```

El método `run()` puede hacer lo que queramos, pero en principio debería insertar uno o más registros en la tabla ***coches*** (usando la *facade* ***DB*** o un modelo *Eloquent*).

A parte de esto, se pueden invocar otros *seeders* desde este método, de tal modo que otras tablas sean rellenadas también (cuidado con las invocaciones circulares). Para ello se utiliza el método `call()` del *seeder*, pasándole un *array* con los nombres *fully qualified* de todas las clases *seeder* deseadas:

```php
public function run()
{
    $this->call([
        UnSeeder::class,
        OtroSeeder::class
    ]);
}
```

Cuando se ejecuta el comando *artisan* sin el *flag* `--class`, se invoca el *seeder* por defecto incluido en *Laravel*: ***DatabaseSeeder***.
