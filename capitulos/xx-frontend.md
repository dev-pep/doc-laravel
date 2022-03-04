# Frontend

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
