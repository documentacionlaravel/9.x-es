# Consola artesanal

- [Introducción](#introduction)
    - [Tinker (REPL)](#tinker)
- [Comandos de escritura](#writing-commands)
    - [Generando comandos](#generating-commands)
    - [Estructura de mando](#command-structure)
    - [Comandos de cierre](#closure-commands)
- [Definición de expectativas de entrada](#defining-input-expectations)
    - [Argumentos](#arguments)
    - [Opciones](#options)
    - [Matrices de entrada](#input-arrays)
    - [Descripciones de entrada](#input-descriptions)
- [E / S de comando](#command-io)
    - [Recuperando entrada](#retrieving-input)
    - [Solicitar información](#prompting-for-input)
    - [Salida de escritura](#writing-output)
- [Registro de comandos](#registering-commands)
- [Ejecución de comandos mediante programación](#programmatically-executing-commands)
    - [Llamada a comandos desde otros comandos](#calling-commands-from-other-commands)
- [Personalización de stub](#stub-customization)
- [Eventos](#events)

<a name="introduction"></a>

## Introducción

Artisan es la interfaz de línea de comandos incluida con Laravel. Artisan existe en la raíz de su aplicación como el `artisan` y proporciona una serie de comandos útiles que pueden ayudarlo mientras construye su aplicación. Para ver una lista de todos los comandos Artisan disponibles, puede usar el comando `list`

```
php artisan list
```

Cada comando también incluye una pantalla de "ayuda" que muestra y describe los argumentos y opciones disponibles del comando. Para ver una pantalla de ayuda, preceda el nombre del comando con `help` :

```
php artisan help migrate
```

<a name="laravel-sail"></a>

#### Vela Laravel

Si está utilizando [Laravel Sail](/docs/%7B%7Bversion%7D%7D/sail) como su entorno de desarrollo local, recuerde usar la `sail` para invocar los comandos de Artisan. Sail ejecutará sus comandos Artisan dentro de los contenedores Docker de su aplicación:

```
./sail artisan list
```

<a name="tinker"></a>

### Tinker (REPL)

Laravel Tinker es un poderoso REPL para el marco de Laravel, impulsado por el paquete [PsySH.](https://github.com/bobthecow/psysh)

<a name="installation"></a>

#### Instalación

Todas las aplicaciones de Laravel incluyen Tinker por defecto. Sin embargo, puede instalar Tinker usando Composer si lo ha eliminado previamente de su aplicación:

```
composer require laravel/tinker
```

> {tip} ¿Busca una interfaz de usuario gráfica para interactuar con su aplicación Laravel? ¡Echa un vistazo a [Tinkerwell](https://tinkerwell.app) !

<a name="usage"></a>

#### Uso

Tinker le permite interactuar con toda su aplicación Laravel en la línea de comando, incluidos sus modelos, trabajos, eventos y más de Eloquent. Para ingresar al entorno de Tinker, ejecute el `tinker` Artisan:

```
php artisan tinker
```

Puede publicar el archivo de configuración de Tinker usando el comando `vendor:publish`

```
php artisan vendor:publish --provider="Laravel\Tinker\TinkerServiceProvider"
```

> {note} La función auxiliar de `dispatch` y el método de `dispatch` `Dispatchable` dependen de la recolección de basura para colocar el trabajo en la cola. Por lo tanto, cuando use tinker, debe usar `Bus::dispatch` o `Queue::push` para enviar trabajos.

<a name="command-allow-list"></a>

#### Lista de permisos de comando

Tinker utiliza una lista de "permisos" para determinar qué comandos de Artisan pueden ejecutarse dentro de su shell. De forma predeterminada, puede ejecutar los comandos `clear-compiled` , `down` , `env` , `inspire` , `migrate` , `optimize` y `up` Si desea permitir más comandos, puede agregarlos a la matriz de `commands` en su archivo de configuración `tinker.php`

```
'commands' => [
    // App\Console\Commands\ExampleCommand::class,
],
```

<a name="classes-that-should-not-be-aliased"></a>

#### Clases que no deben tener alias

Normalmente, Tinker asigna automáticamente un alias a las clases cuando interactúas con ellas en Tinker. Sin embargo, es posible que desee nunca asignar un alias a algunas clases. Puede lograr esto enumerando las clases en la `dont_alias` de su archivo de configuración `tinker.php`

```
'dont_alias' => [
    App\Models\User::class,
],
```

<a name="writing-commands"></a>

## Comandos de escritura

Además de los comandos proporcionados con Artisan, puede crear sus propios comandos personalizados. Los comandos se almacenan normalmente en el directorio `app/Console/Commands` sin embargo, puede elegir su propia ubicación de almacenamiento siempre que Composer pueda cargar sus comandos.

<a name="generating-commands"></a>

### Generando comandos

Para crear un nuevo comando, puede usar el `make:command` Artisan. Este comando creará una nueva clase de comando en el directorio `app/Console/Commands` No se preocupe si este directorio no existe en su aplicación; se creará la primera vez que ejecute el `make:command` Artisan:

```
php artisan make:command SendEmails
```

<a name="command-structure"></a>

### Estructura de mando

Después de generar su comando, debe definir los valores apropiados para las `signature` y `description` de la clase. Estas propiedades se utilizarán al mostrar su comando en la pantalla de `list` La `signature` también le permite definir [las expectativas de entrada de su comando](#defining-input-expectations) . Se `handle` cuando se ejecute su comando. Puede colocar su lógica de comando en este método.

Echemos un vistazo a un comando de ejemplo. Tenga en cuenta que podemos solicitar cualquier dependencia que necesitemos a través del método `handle` El [contenedor de servicios de](/docs/%7B%7Bversion%7D%7D/container) Laravel inyectará automáticamente todas las dependencias que están insinuadas en la firma de este método:

```
<?php

namespace App\Console\Commands;

use App\Models\User;
use App\Support\DripEmailer;
use Illuminate\Console\Command;

class SendEmails extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'mail:send {user}';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Send a marketing email to a user';

    /**
     * Create a new command instance.
     *
     * @return void
     */
    public function __construct()
    {
        parent::__construct();
    }

    /**
     * Execute the console command.
     *
     * @param  \App\Support\DripEmailer  $drip
        * @return mixed
     */
    public function handle(DripEmailer $drip)
    {
        $drip->send(User::find($this->argument('user')));
    }
}
```

> {consejo} Para una mayor reutilización del código, es una buena práctica mantener los comandos de la consola ligeros y dejar que se dediquen a los servicios de la aplicación para realizar sus tareas. En el ejemplo anterior, observe que inyectamos una clase de servicio para hacer el "trabajo pesado" de enviar los correos electrónicos.

<a name="closure-commands"></a>

### Comandos de cierre

Los comandos basados en cierres proporcionan una alternativa a la definición de comandos de consola como clases. De la misma manera que los cierres de rutas son una alternativa a los controladores, piense en los cierres de comandos como una alternativa a las clases de comandos. Dentro del método de `commands` `app/Console/Kernel.php` , Laravel carga el archivo `routes/console.php`

```
/**
 * Register the closure based commands for the application.
 *
 * @return void
 */
protected function commands()
{
    require base_path('routes/console.php');
}
```

Aunque este archivo no define rutas HTTP, define puntos de entrada (rutas) basados en la consola en su aplicación. Dentro de este archivo, puede definir todos sus comandos de consola basados en el cierre usando el método `Artisan::command` El `command` acepta dos argumentos: la [firma](#defining-input-expectations) del comando y un cierre que recibe los argumentos y opciones del comando:

```
Artisan::command('mail:send {user}', function ($user) {
    $this->info("Sending email to: {$user}!");
});
```

El cierre está vinculado a la instancia de comando subyacente, por lo que tiene acceso completo a todos los métodos auxiliares a los que normalmente podría acceder en una clase de comando completa.

<a name="type-hinting-dependencies"></a>

#### Dependencias de sugerencias de tipo

Además de recibir los argumentos y las opciones de su comando, los cierres de comandos también pueden indicar dependencias adicionales que le gustaría resolver fuera del [contenedor de servicios](/docs/%7B%7Bversion%7D%7D/container) :

```
use App\Models\User;
use App\Support\DripEmailer;

Artisan::command('mail:send {user}', function (DripEmailer $drip, $user) {
    $drip->send(User::find($user));
});
```

<a name="closure-command-descriptions"></a>

#### Descripciones de los comandos de cierre

Al definir un comando basado en cierre, puede usar el `purpose` para agregar una descripción al comando. Esta descripción se mostrará cuando ejecute los comandos `php artisan list` o `php artisan help`

```
Artisan::command('mail:send {user}', function ($user) {
    // ...
})->purpose('Send a marketing email to a user');
```

<a name="defining-input-expectations"></a>

## Definición de expectativas de entrada

Al escribir comandos de consola, es común recopilar información del usuario a través de argumentos u opciones. Laravel hace que sea muy conveniente definir la entrada que espera del usuario usando la `signature` en sus comandos. La `signature` permite definir el nombre, los argumentos y las opciones del comando en una sintaxis única, expresiva y similar a una ruta.

<a name="arguments"></a>

### Argumentos

Todos los argumentos y opciones proporcionados por el usuario están entre llaves. En el siguiente ejemplo, el comando define un argumento obligatorio: `user` :

```
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user}';
```

También puede hacer que los argumentos sean opcionales o definir valores predeterminados para los argumentos:

```
// Optional argument...
mail:send {user?}

// Optional argument with default value...
mail:send {user=foo}
```

<a name="options"></a>

### Opciones

Las opciones, como los argumentos, son otra forma de entrada del usuario. Las opciones tienen como prefijo dos guiones ( `--` ) cuando se proporcionan a través de la línea de comandos. Hay dos tipos de opciones: las que reciben un valor y las que no. Las opciones que no reciben un valor sirven como un "cambio" booleano. Echemos un vistazo a un ejemplo de este tipo de opción:

```
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue}';
```

En este ejemplo, el `--queue` se puede especificar al llamar al comando Artisan. Si se `--queue` interruptor --queue, el valor de la opción será `true` . De lo contrario, el valor será `false` :

```
php artisan mail:send 1 --queue
```

<a name="options-with-values"></a>

#### Opciones con valores

A continuación, echemos un vistazo a una opción que espera un valor. Si el usuario debe especificar un valor para una opción, debe agregar un sufijo al nombre de la opción con un signo `=`

```
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue=}';
```

En este ejemplo, el usuario puede pasar un valor para la opción así. Si no se especifica la opción al invocar el comando, su valor será `null` :

```
php artisan mail:send 1 --queue=default
```

Puede asignar valores predeterminados a las opciones especificando el valor predeterminado después del nombre de la opción. Si el usuario no pasa ningún valor de opción, se utilizará el valor predeterminado:

```
mail:send {user} {--queue=default}
```

<a name="option-shortcuts"></a>

#### Accesos directos de opciones

Para asignar un atajo al definir una opción, puede especificarlo antes del nombre de la opción y usar el `|` carácter como delimitador para separar el acceso directo del nombre completo de la opción:

```
mail:send {user} {--Q|queue}
```

<a name="input-arrays"></a>

### Matrices de entrada

Si desea definir argumentos u opciones para esperar múltiples valores de entrada, puede usar el carácter `*` Primero, echemos un vistazo a un ejemplo que especifica tal argumento:

```
mail:send {user*}
```

Al llamar a este método, los `user` se pueden pasar en orden a la línea de comando. Por ejemplo, el siguiente comando establecerá el valor de `user` en una matriz con `foo` y `bar` como sus valores:

```
php artisan mail:send foo bar
```

<a name="option-arrays"></a>

#### Matrices de opciones

Al definir una opción que espera múltiples valores de entrada, cada valor de opción pasado al comando debe tener como prefijo el nombre de la opción:

```
mail:send {user} {--id=*}

php artisan mail:send --id=1 --id=2
```

<a name="input-descriptions"></a>

### Descripciones de entrada

Puede asignar descripciones a los argumentos de entrada y las opciones separando el nombre del argumento de la descripción con dos puntos. Si necesita un poco más de espacio para definir su comando, no dude en difundir la definición en varias líneas:

```
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send
                        {user : The ID of the user}
                        {--queue= : Whether the job should be queued}';
```

<a name="command-io"></a>

## E / S de comando

<a name="retrieving-input"></a>

### Recuperando entrada

Mientras se ejecuta su comando, es probable que necesite acceder a los valores de los argumentos y opciones aceptados por su comando. Para hacerlo, puede utilizar los métodos de `argument` y `option` Si un argumento u opción no existe, se devolverá un `null`

```
/**
 * Execute the console command.
 *
 * @return int
 */
public function handle()
{
    $userId = $this->argument('user');

    //
}
```

Si necesita recuperar todos los argumentos como una `array` , llame al método de `arguments`

```
$arguments = $this->arguments();
```

Las opciones se pueden recuperar con la misma facilidad que los argumentos utilizando el método de `option` Para recuperar todas las opciones como una matriz, llame al método de `options`

```
// Retrieve a specific option...
$queueName = $this->option('queue');

// Retrieve all options as an array...
$options = $this->options();
```

<a name="prompting-for-input"></a>

### Solicitar información

Además de mostrar la salida, también puede pedirle al usuario que proporcione información durante la ejecución de su comando. El `ask` le pedirá al usuario la pregunta dada, aceptará su entrada y luego devolverá la entrada del usuario a su comando:

```
/**
 * Execute the console command.
 *
 * @return mixed
 */
public function handle()
{
    $name = $this->ask('What is your name?');
}
```

El `secret` es similar a `ask` , pero la entrada del usuario no será visible para ellos mientras escribe en la consola. Este método es útil cuando se solicita información confidencial, como contraseñas:

```
$password = $this->secret('What is the password?');
```

<a name="asking-for-confirmation"></a>

#### Pedir confirmación

Si necesita pedirle al usuario una simple confirmación de "sí o no", puede utilizar el método de `confirm` De forma predeterminada, este método devolverá `false` . Sin embargo, si el usuario ingresa `y` `yes` en respuesta a la solicitud, el método devolverá `true` .

```
if ($this->confirm('Do you wish to continue?')) {
    //
}
```

Si es necesario, puede especificar que la solicitud de confirmación debe devolver `true` por defecto pasando `true` como segundo argumento del método de `confirm`

```
if ($this->confirm('Do you wish to continue?', true)) {
    //
}
```

<a name="auto-completion"></a>

#### Autocompletar

El `anticipate` se puede utilizar para proporcionar autocompletar para posibles opciones. El usuario aún puede proporcionar cualquier respuesta, independientemente de las sugerencias de autocompletar:

```
$name = $this->anticipate('What is your name?', ['Taylor', 'Dayle']);
```

Alternativamente, puede pasar un cierre como segundo argumento del método `anticipate` El cierre se llamará cada vez que el usuario escriba un carácter de entrada. El cierre debe aceptar un parámetro de cadena que contenga la entrada del usuario hasta el momento y devolver una matriz de opciones para autocompletar:

```
$name = $this->anticipate('What is your address?', function ($input) {
    // Return auto-completion options...
});
```

<a name="multiple-choice-questions"></a>

#### Preguntas de respuestas múltiples

Si necesita darle al usuario un conjunto predefinido de opciones al hacer una pregunta, puede usar el método de `choice` Puede configurar el índice de matriz del valor predeterminado que se devolverá si no se elige ninguna opción pasando el índice como tercer argumento del método:

```
$name = $this->choice(
    'What is your name?',
    ['Taylor', 'Dayle'],
    $defaultIndex
);
```

Además, el `choice` acepta argumentos cuarto y quinto opcionales para determinar el número máximo de intentos para seleccionar una respuesta válida y si se permiten selecciones múltiples:

```
$name = $this->choice(
    'What is your name?',
    ['Taylor', 'Dayle'],
    $defaultIndex,
    $maxAttempts = null,
    $allowMultipleSelections = false
);
```

<a name="writing-output"></a>

### Salida de escritura

Para enviar la salida a la consola, puede usar los métodos de `line` , `info` , `comment` , `question` y `error` Cada uno de estos métodos utilizará colores ANSI apropiados para su propósito. Por ejemplo, mostremos información general al usuario. Normalmente, el `info` se mostrará en la consola como texto de color verde:

```
/**
 * Execute the console command.
 *
 * @return mixed
 */
public function handle()
{
    // ...

    $this->info('The command was successful!');
}
```

Para mostrar un mensaje de error, utilice el método de `error` El texto del mensaje de error generalmente se muestra en rojo:

```
$this->error('Something went wrong!');
```

Puede utilizar el `line` para mostrar texto sin formato y sin color:

```
$this->line('Display this on the screen');
```

Puede utilizar el `newLine` para mostrar una línea en blanco:

```
// Write a single blank line...
$this->newLine();

// Write three blank lines...
$this->newLine(3);
```

<a name="tables"></a>

#### Mesas

El `table` facilita el formato correcto de varias filas / columnas de datos. Todo lo que necesita hacer es proporcionar los nombres de las columnas y los datos para la tabla y Laravel calculará automáticamente el ancho y alto apropiados de la tabla para usted:

```
use App\Models\User;

$this->table(
    ['Name', 'Email'],
    User::all(['name', 'email'])->toArray()
);
```

<a name="progress-bars"></a>

#### Barras de progreso

Para las tareas de ejecución prolongada, puede resultar útil mostrar una barra de progreso que informe a los usuarios qué tan completa está la tarea. Usando el `withProgressBar` , Laravel mostrará una barra de progreso y avanzará su progreso para cada iteración sobre un valor iterable dado:

```
use App\Models\User;

$users = $this->withProgressBar(User::all(), function ($user) {
    $this->performTask($user);
});
```

A veces, es posible que necesite más control manual sobre cómo avanza una barra de progreso. Primero, defina el número total de pasos que recorrerá el proceso. Luego, avance la barra de progreso después de procesar cada elemento:

```
$users = App\Models\User::all();

$bar = $this->output->createProgressBar(count($users));

$bar->start();

foreach ($users as $user) {
    $this->performTask($user);

    $bar->advance();
}

$bar->finish();
```

> {tip} Para obtener opciones más avanzadas, consulte la [documentación del componente de la barra de progreso de Symfony](https://symfony.com/doc/current/components/console/helpers/progressbar.html) .

<a name="registering-commands"></a>

## Registro de comandos

Todos los comandos de su consola están registrados dentro de la `App\Console\Kernel` su aplicación, que es el "kernel de la consola" de su aplicación. Dentro del `commands` de esta clase, verá una llamada al método de `load` El método de `load` `app/Console/Commands` y registrará automáticamente cada comando que contiene con Artisan. Incluso puede realizar llamadas adicionales al `load` para escanear otros directorios en busca de comandos Artisan:

```
/**
 * Register the commands for the application.
 *
 * @return void
 */
protected function commands()
{
    $this->load(__DIR__.'/Commands');
    $this->load(__DIR__.'/../Domain/Orders/Commands');

    // ...
}
```

Si es necesario, puede registrar comandos manualmente agregando el nombre de la clase del comando a la `$commands` de su clase `App\Console\Kernel` Cuando Artisan arranca, todos los comandos enumerados en esta propiedad serán resueltos por el [contenedor de servicios](/docs/%7B%7Bversion%7D%7D/container) y registrados con Artisan:

```
protected $commands = [
    Commands\SendEmails::class
];
```

<a name="programmatically-executing-commands"></a>

## Ejecución de comandos mediante programación

A veces, es posible que desee ejecutar un comando Artisan fuera de la CLI. Por ejemplo, es posible que desee ejecutar un comando Artisan desde una ruta o controlador. Puede usar el método de `call` `Artisan` para lograr esto. El `call` acepta el nombre de la firma del comando o el nombre de la clase como primer argumento, y una matriz de parámetros del comando como segundo argumento. Se devolverá el código de salida:

```
use Illuminate\Support\Facades\Artisan;

Route::post('/user/{user}/mail', function ($user) {
    $exitCode = Artisan::call('mail:send', [
        'user' => $user, '--queue' => 'default'
    ]);

    //
});
```

Alternativamente, puede pasar todo el comando Artisan al `call` como una cadena:

```
Artisan::call('mail:send 1 --queue=default');
```

<a name="passing-array-values"></a>

#### Pasar valores de matriz

Si su comando define una opción que acepta una matriz, puede pasar una matriz de valores a esa opción:

```
use Illuminate\Support\Facades\Artisan;

Route::post('/mail', function () {
    $exitCode = Artisan::call('mail:send', [
        '--id' => [5, 13]
    ]);
});
```

<a name="passing-boolean-values"></a>

#### Pasar valores booleanos

Si necesita especificar el valor de una opción que no acepta valores de cadena, como la `--force` en el `migrate:refresh` , debe pasar `true` o `false` como valor de la opción:

```
$exitCode = Artisan::call('migrate:refresh', [
    '--force' => true,
]);
```

<a name="queueing-artisan-commands"></a>

#### Colas de comandos artesanales

Usando el método de `queue` `Artisan` , puede incluso poner en cola los comandos Artisan para que los [trabajadores de la cola los](/docs/%7B%7Bversion%7D%7D/queues) procesen en segundo plano. Antes de usar este método, asegúrese de haber configurado su cola y está ejecutando un detector de cola:

```
use Illuminate\Support\Facades\Artisan;

Route::post('/user/{user}/mail', function ($user) {
    Artisan::queue('mail:send', [
        'user' => $user, '--queue' => 'default'
    ]);

    //
});
```

Con los `onConnection` y `onQueue` , puede especificar la conexión o cola a la que se debe enviar el comando Artisan:

```
Artisan::queue('mail:send', [
    'user' => 1, '--queue' => 'default'
])->onConnection('redis')->onQueue('commands');
```

<a name="calling-commands-from-other-commands"></a>

### Llamada a comandos desde otros comandos

A veces, es posible que desee llamar a otros comandos desde un comando Artisan existente. Puede hacerlo utilizando el método de `call` Este `call` acepta el nombre del comando y una matriz de argumentos / opciones del comando:

```
/**
 * Execute the console command.
 *
 * @return mixed
 */
public function handle()
{
    $this->call('mail:send', [
        'user' => 1, '--queue' => 'default'
    ]);

    //
}
```

Si desea llamar a otro comando de consola y suprimir toda su salida, puede usar el método `callSilently` El `callSilently` tiene la misma firma que el método de `call`

```
$this->callSilently('mail:send', [
    'user' => 1, '--queue' => 'default'
]);
```

<a name="stub-customization"></a>

## Personalización de stub

`make` la consola Artisan se utilizan para crear una variedad de clases, como controladores, trabajos, migraciones y pruebas. Estas clases se generan utilizando archivos "stub" que se rellenan con valores basados en su entrada. Sin embargo, es posible que desee realizar pequeños cambios en los archivos generados por Artisan. Para lograr esto, puede usar el `stub:publish` para publicar los stubs más comunes en su aplicación para que pueda personalizarlos:

```
php artisan stub:publish
```

Los stubs publicados se ubicarán dentro de un `stubs` en la raíz de su aplicación. Cualquier cambio que realice en estos stubs se reflejará cuando genere sus clases correspondientes utilizando los comandos `make`

<a name="events"></a>

## Eventos

Artisan distribuye tres eventos cuando ejecuta comandos: `Illuminate\Console\Events\ArtisanStarting` , `Illuminate\Console\Events\CommandStarting` e `Illuminate\Console\Events\CommandFinished` . El `ArtisanStarting` se envía inmediatamente cuando Artisan comienza a ejecutarse. A continuación, el `CommandStarting` se distribuye inmediatamente antes de que se ejecute un comando. Finalmente, el `CommandFinished` se distribuye una vez que finaliza la ejecución de un comando.
