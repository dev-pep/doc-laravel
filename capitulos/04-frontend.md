# Frontend

## Plantillas *Blade*

Permite extender un archivo *HTML* (incluyendo también, si se desea, fragmentos *PHP*) utilizando **directivas *Blade***.

Las vistas se almacenan en ***resources/views***. Tienen extensión ***.blade.php***.

### Extensión de plantillas

`@extends` sirve para heredar de una plantilla base. Por ejemplo `@extends('layout.app')` extenderá la plantilla ***resources/views/layout/app.blade.php***.

Esto es incluye la plantilla base en nuestro archivo *Blade*. Pero para personalizar el contenido, debemos definir una serir de **secciones**, o lugares donde la plantilla espera que se le pase contenido. Las secciones las definimos con la directiva `@section()`, y la plantilla base incluye ese contenido mediante la directiva `@yield()`.

Supongamos que en un punto del código de la plantilla base, se incluye `@yield('titulo')`. Cuando nosotros extendamos esa plantilla base, deberemos definir el contenido de la sección ***titulo***. Esto se podría hacer así:

```
@section('titulo', 'Página de bienvenida')
```

Esto simplemente sustituye la llamada a `@yield()` con el texto ***Página de bienvenida***.

Si queremos expandir varias líneas en lugar de un simple *string*, disponemos de esta sintaxis alternativa:

```
@section('display')
    <h1>Esta es la sección display</h1>
    <!-- Más contenido para la sección -->
@endsection
```

Es posible también que la plantilla base tenga alguna sección definida (puede a su vez extender otra plantilla, o no). Si definimos una sección con el mismo nombre que la de la plantilla base, nuestro contenido sustituirá a la contenido de la sección del padre. Sin embargo, dentro de la definición de nuestra sección podemos incluir la directiva `@parent`, lo cual se expandirá en el contenido de la sección base.

En el caso de que la plantilla base defina una sección, si esta plantilla no extiende otra plantilla, la sección quedaría definida únicamente, y no se mostraría. Si queremos que la sección que definimos quede *yielded* inmediatamente, en lugar de terminarla con `@endsection` la terminaremos con `@show`.

La directiva `@yield()` acepta un segundo argumento opcional, que usará si no encuentra la sección especificada.

### Parámetros

Es posible pasar argumentos a una vista. Por ejemplo, si definimos la ruta:

```php
Route::get('greeting', function() {
    return view('welcome', ['name' => 'Samantha']);
});
```

Entonces podremos usar el contenido de la variable dentro de nuestra plantilla mediante `{{ $name }}`.

Las dobles llaves son la sentencia *echo* de *Blade*, con lo que se puede incluir cualquier código *PHP* en su interior.

Para saber si una variable ha sido definida, *Blade* tiene la directiva `@isset($variable) ... @endisset` (puede incluir `@else`, como cualquier directiva condicional).

### Estructuras de control

*Blade* dispone de `@if()`, `@elseif()`, `@else`, y `@endif`, a las que se les debe pasar una expresión *PHP*:

```
@if(count($records) < 10)
    Tengo poquitos.
@elseif(count($records) == 10)
    Tengo diez.
@else
    Tengo muchos.
@endif
```

Además, existe la directiva `@unless()` ... `@endunless`, que es lo contrario de `@if()`.

También se dispone de las directivas `@isset()` ... `@endisset` y `@empty()` ... `@endempty` que son verdaderas si la variable del argumento tiene valor y si el *array* está vacío, respectivamente.

El código de la directiva `@auth` ... `@endauth` se utilizará si el usuario está autenticado, todo lo contrario que `@guest` ... `@endguest`.

Para comprobar si una sección tiene contenido, `@hasSection()` ... `@endif`, pasándole como argumento el nombre de la sección. La inversa es `@sectionMissing()` ... `@endif`.

Para un mecanismo *switch*:

```
@switch($variable)
    @case(1)
        Primer caso...
        @break
    @case(10)
        Segundo caso...
        @break
    ...
    @default
        Caso default...
@endswitch
```

En cuanto a bucles:

```
@for ($i = 0; $i < 10; $i++)
    El valor es {{ $i }}
@endfor

@foreach ($users as $user)
    <p>Este es el usuario {{ $user->id }}</p>
@endforeach

@forelse ($users as $user)
    <li>{{ $user->name }}</li>
@empty
    <p>No users</p>
@endforelse

@while (true)
    <p>Looping forever</p>
@endwhile
```

Dentro de los bucles disponemos de las directivas `@break` y `@continue`, que son equivalentes al funcionamiento de sus contrapates en *PHP*. Pero además admiten una condición, de tal modo que la directiva se ejecuta solo en el caso que la condición sea verdadera: `@continue($i < 1)`, o `@break($i > 100)`.

Dentro de un bucle tenemos acceso a la variable `$loop`, que nos da informaciones sobre el estado del bucle. `$loop->first` es verdadero si estamos en la primera iteración, mientras que `$loop->last` es verdadero si estamos en la última.

Si el bucle está anidado, `$loop->parent` es la variable `$loop` del bucle padre.

`$loop->index` es el índice de la iteración actual (empieza en 0). `$loop->iteration` es la iteración actual (empieza en 1). `$loop->remaining` es la cantidad de iteraciones que quedan para terminar. `$loop->count` es el número de elementos del objeto que estamos iterando. `$loop->depth` indica el nivel de anidamiento del bucle.

### Comentarios

Se incluyen entre `{{--` y `--}}`, y no se insertan en el *HTML* resultante.

### *PHP*

Es posible insertar bloques de código *PHP* usando las directivas `@php` ... `@endphp`.

### Formularios

Para que el *submit* de un formulario sea admitido se debe generar un campo oculto con un *token* usado para la protección *CSRF*. Para ello hay que incluir siempre la directiva `@csrf` dentro de los formularios.

Si queremos que un formulario realice una petición *HTTP* ***PUT***, ***PATCH*** o ***DELETE***, deberemos incluir en el formulario una directiva `@method()` con el método deseado (por ejemplo, `@method('PUT')`). Esto añade un campo oculto que hará que *Laravel* trate esa petición como si fuese del método indicado, y no del método efectivo del formulario.

### Inclusión de subvistas

Mediante la directiva `@include()` se puede incluir una vista dentro de la actual. Todas las variables que estén disponibles en la plantilla actual, lo estarán también en la plantilla incluida. Se le pasa como argumento el nombre de la plantilla a incluir (con *dot notation*, como siempre). Como segundo parámetro se le puede pasar opcionalmente un *array* con parámetros para la plantilla a incluir.

Si la plantilla no existe, se levantará un error. Para evitarlo, se puede incluir mediante `@includeIf()`, la cual no levantará error si la plantilla especificada no existe.

La directiva `@includeWhen()` es como `@include()`, pero se inserta un primer parámetro con una condición, la cual, de evaluarse verdadera, realiza la acción de inclusión. Si la expresión no es verdadera, no se incluye la plantilla. Esta directiva es la contraria de `@includeUnless()`.

`@includeFirst()` es como `@include()`, pero en lugar de pasársele un nombre de plantilla, se le pasa un *array* con nombres de plantilla. Se incluirá la primera de ellas que exista.

## Localización

Soporte multilenguage incorporado en *Laravel*.

Los *strings* de idiomas están en ***resources/lang***. En este directorio deben ubicarse las carpetas de cada lenguage: ***en***, ***es***, etc. En estas carpetas se almacenan los archivos de *strings*.

Lo único que hacen los archivos de *strings* es retornar un *array* cuyos elementos tienen como clave el identificador del *string*, y como valor, el contenido del mismo.

El código identificador de un *locale* concreto debe seguir la nomenclatura de ISO 15897, por lo que si es un lenguaje con diferencias territoriales, el formato es del tipo ***en_GB*** (no ***en-gb***).

El lenguaje por defecto está definido en ***config/app.php***, pero se puede establecer en *run time* mediante la *facade* ***App***:

```php
Route::get('welcome/{locale}', function($locale) {
    if(in_array($locale, ['en', 'es', 'fr']))
        App::setLocale($locale);
});
```

También hay un *fallback locale*, definido en ***config/app.php***, cuyos *strings* se usan cuando no existe traducción de un *string* en el *locale* actual.

Para saber cuál es el *locale* actual, se usa `App::getLocale()`. Para saber si el *locale* actual se corresponde con un locale específico, la comparación se puede usar:

```php
if(App::isLocale('en')) ...
```

### Definición de los *strings*

Se deben crear los archivos deseados dentro de los directorios de cada *locale*. Cada archivo debe retornar un *array* con la clave (*ID* o *short key* de cada *string*).

Por ejemplo, en ***resources/lang/es/mensajes.php*** podríamos tener:

```
<?php
    return [
        'bienvenida' => 'Bienvenido a nuestra aplicación.',
        'gracias' => 'Gracias por visitar nuestra web.'
    ];
```

En la el fichero de traducción al inglés (***resources/lang/en/mensajes.php***), tendríamos:

```
<?php
    return [
        'bienvenida' => 'Welcome to our application.',
        'gracias' => 'Thanks for visiting our website.'
    ];
```

Y el acceso a los *strings* se haría de la siguiente forma:

```php
echo __('mensajes.bienvenida');
```

En este caso, la función `__()` retornará el mensaje de bienvenida en el idioma del *locale* actual (si lo hemos definido para ese *locale*).

Dentro de una plantilla *Blade*, pueden incluirse *strings* de dos formas (equivalentes):

```
{{ __('mensajes.bienvenida') }}
```

```
@lang('mensajes.bienvenida')
```

Cuando las aplicaciones se vuelven grandes, guardar las traducciones mediante *short keys* se puede llegar a hacer muy confuso. Es por ello que también se pueden definir las traducciones utilizando como clave la "traducción por defecto". Estos archivos se almacenan como *JSON* en ***resources/lang*** directamente, y tendrán como nombre el nombre del *locale* (y extensión *json*). Por ejemplo, el archivo ***resources/lang/en.json*** podría contener:

```json
{
    "Bienvenido a nuestra aplicación.": "Welcome to our application",
    "Gracias por visitar nuestra web.": "Thanks for visiting our website."
}
```

En este caso, el acceso al *string* se hace igual:

```php
echo __('Bienvenido a nuestra aplicación.');
```

Pueden definirse parámetros en los *strings*, prefijando dos puntos (***:***):

```php
'bienvenida' => 'Bienvenido a nuestra aplicación, :nombre.',
```

A la hora de acceder al *string* se debe añadir un *array* con los argumentos:

```php
echo __('bienvenida', ['nombre' => 'Doyle']);
```

Si en la definición del *string* el *placeholder* del parámetro está todo en mayúsculas o tiene únicamente la primera letra en mayúsculas, el valor que se le pase será cambiado para coincidir con esa capitalización.

Para traducir *strings* de paquetes que tengan sus propios *language files*, en lugar de editar esos ficheros del paquete, se pueden *override* creando *language files* en ***resources/lang/vendor/\<paquete>/\<locale>/\<archivo>***.

Por ejemplo, supongamos que el paquete ***skyrim/hearthfire*** (recordemos que esto es *vendor/paquete*) tiene un archivo de lenguaje en inglés llamado ***messages.php***. Entonces solo tenemos que crear un archivo ***resources/lang/vendor/hearthfire/es/messages.php*** con nuestros propios *strings*.

## Frontend Scaffolding

Por defecto, *Laravel* permite trabajar con los paquetes *frontend* ***Bootstrat***, ***React*** y/o ***Vue***.

Para generar una estructura con estos paquetes, es necesario instalar en nuestra aplicación el paquete ***laravel/ui*** mediante *Composer*:

```
composer require laravel/ui:^2.4
```

Una vez hecho esto, podemos generar estas estructuras desde ***artisan***. A elegir:

```
php artisan ui bootstrap
php artisan ui vue
php artisan ui react
```

Si a este comando le añadimos `--auth`, se incluirá el mecanismo de autenticación de usuarios en esta estructura.
