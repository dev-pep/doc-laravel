# Frontend

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

En el fichero de traducción al inglés (***resources/lang/en/mensajes.php***), tendríamos:

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

Por defecto, *Laravel* permite trabajar con los paquetes *frontend* ***Bootstrap***, ***React*** y/o ***Vue***.

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
