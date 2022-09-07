<p align="center">Документация <a href="https://github.com/Velman04/laravel-centrifugo-broadcaster/blob/master/README.md">EN</a> | <b>RU</b></p>

<p align="center">
<a href="https://github.com/Velman04/laravel-centrifugo-broadcaster/releases"><img src="https://img.shields.io/github/release/Velman04/laravel-centrifugo-broadcaster.svg?style=flat-square" alt="Latest Version"></a>
<a href="https://github.styleci.io/repos/372425291?branch=master"><img src="https://github.styleci.io/repos/372425291/shield?branch=master" alt="StyleCI"></a>
<a href="https://scrutinizer-ci.com/g/Velman04/laravel-centrifugo-broadcaster/?branch=master"><img src="https://scrutinizer-ci.com/g/Velman04/laravel-centrifugo-broadcaster/badges/quality-score.png?b=master" alt="StyleCI"></a>
<a href="https://packagist.org/packages/Velman04/laravel-centrifugo-broadcaster"><img src="https://img.shields.io/packagist/dt/Velman04/laravel-centrifugo-broadcaster.svg?style=flat-square" alt="Total Downloads"></a>
<a href="https://github.com/Velman04/laravel-centrifugo-broadcaster/blob/master/LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="Software License"></a>
</p>

<h1 align="center">Laravel Centrifugo Broadcaster</h1>
<h2 align="center">Centrifugo broadcast driver for Laravel 8.75 - 9.x </h2>

## Introduction

Centrifugo broadcaster for laravel is fork of [laravel-centrifuge](https://github.com/denis660/laravel-centrifuge),
based on:

- [LaraComponents/centrifuge-broadcaster](https://github.com/LaraComponents/centrifuge-broadcaster)
- [centrifugal/phpcent](https://github.com/centrifugal/phpcent)

## Features

- Compatible with latest [Centrifugo 4.0.0](https://github.com/centrifugal/centrifugo/) 🚀
- Wrapper over [Centrifugo HTTP API](https://centrifugal.github.io/centrifugo/server/http_api/) 🔌
- Authentication with JWT token (HMAC algorithm) for anonymous, authenticated user and private channel 🗝️

## Requirements

- PHP >= 7.4
- Laravel 8.75 - 9.x
- guzzlehttp/guzzle 6 - 7
- Centrifugo Server 4.0.0 or newer (see [here](https://github.com/centrifugal/centrifugo))

## Installation

Require this package with composer:

```bash
composer req Velman04/laravel-centrifugo-broadcaster
```

Open your `config/app.php` and add the following to the providers array:

```php
return [
    'providers' => [
        // Add service provider ( Laravel 5.4 or below )
        Opekunov\Centrifugo\CentrifugoServiceProvider::class,
    
        // And uncomment BroadcastServiceProvider
        App\Providers\BroadcastServiceProvider::class,
    ],
];
```

Open your `config/broadcasting.php` and add new connection like this:

```php
return [
        'centrifugo' => [
            'driver' => 'centrifugo',
            'secret'  => env('CENTRIFUGO_SECRET'),
            'apikey'  => env('CENTRIFUGO_APIKEY'),
            'api_path' => env('CENTRIFUGO_API_PATH', '/api'), // Centrifugo api endpoint (default '/api')
            'url'     => env('CENTRIFUGO_URL', 'http://localhost:8000'), // centrifugo api url
            'verify'  => env('CENTRIFUGO_VERIFY', false), // Verify host ssl if centrifugo uses this
            'ssl_key' => env('CENTRIFUGO_SSL_KEY', null), // Self-Signed SSl Key for Host (require verify=true),
            'show_node_info' => env('CENTRIFUGO_SHOW_NODE_INFO', false), // Show node info in response with auth token
            'timeout' => env('CENTRIFUGO_TIMEOUT', 3), // Float describing the total timeout of the request to centrifugo api in seconds. Use 0 to wait indefinitely (the default is 3)
            'tries' => env('CENTRIFUGO_TRIES', 1) //Number of times to repeat the request, in case of failure (the default is 1)
        ],
];
```

Also you should add these two lines to your `.env` file:

```
CENTRIFUGO_SECRET=token_hmac_secret_key-from-centrifugo-config
CENTRIFUGO_APIKEY=api_key-from-centrifugo-config
CENTRIFUGO_URL=http://localhost:8000
```

These lines are optional:

```
CENTRIFUGO_SSL_KEY=/etc/ssl/some.pem
CENTRIFUGO_VERIFY=false
CENTRIFUGO_API_PATH=/api
CENTRIFUGO_SHOW_NODE_INFO=false
CENTRIFUGO_TIMEOUT=10
CENTRIFUGO_TRIES=1
```

Don't forget to change `BROADCAST_DRIVER` setting in .env file!

```
BROADCAST_DRIVER=centrifugo
```

## Basic Usage

To configure Centrifugo server, read [official documentation](https://centrifugal.dev/)

For broadcasting events, see [official documentation of laravel](https://laravel.com/docs/9.x/broadcasting)

### Open your `app/Http/Middleware/VerifyCsrfToken.php` and add the following to the except array:
```php
protected $except = [
    '/centrifuge/*'
];
```

### Authentication example:
```php
// routes/web.php
use App\Http\Controllers\Centrifuge\{
    ClientConnectionToken as CentrifugeClientConnectionToken,
    ChannelConnectionToken as CentrifugeChannelConnectionToken
};

// Centrifugal
Route::prefix('centrifuge')
    ->name('centrifuge.')
    ->group(function () {
        Route::post('client-connection-token', CentrifugeClientConnectionToken::class);
        Route::post('channel-connection-token', CentrifugeChannelConnectionToken::class);
    });
```

### Basic controller for route centrifuge:
Command for create controller: `php artisan make:controller CentrifugeBaseController`
```php
namespace App\Http\Controllers\Centrifuge;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Carbon;

class CentrifugeBaseController extends Controller
{
    public function userId(Request $request): int
    {
        return $request->user()->id ?? 0;
    }

    public function channel(Request $request): string|null
    {
        return $request->input('channel', null);
    }

    public function tokenValidityPeriod(): Carbon
    {
        return now()->addDay()->endOfDay();
    }

    public function email(Request $request): string
    {
        return $request->user()->email ?? 'Guest';
    }
}
```

### Client connection token controller:
Command for create controller: `php artisan make:controller Centrifuge/ClientConnectionToken`
```php
namespace App\Http\Controllers\Centrifuge;

use Illuminate\Http\{Request, JsonResponse};
use Opekunov\Centrifugo\Centrifugo;

class ClientConnectionToken extends CentrifugeBaseController
{
    public function __invoke(Request $request, Centrifugo $centrifugo): JsonResponse
    {
        return response()->json([
            'token' => $centrifugo->generateConnectionToken($this->userId($request), 0, [
                'email' => $this->email($request),
            ]),
        ]);
    }
}
```

### Channel connection token controller:
Command for create controller: `php artisan make:controller Centrifuge/ChannelConnectionToken`
```php
namespace App\Http\Controllers\Centrifuge;

use Illuminate\Http\{Request, JsonResponse};
use Opekunov\Centrifugo\Centrifugo;

class ChannelConnectionToken extends CentrifugeBaseController
{
    public function __invoke(Request $request, Centrifugo $centrifugo): JsonResponse
    {
        if ($request->has('channel')) {
            $token = $centrifugo->generatePrivateChannelToken($this->userId($request), $this->channel($request), $this->tokenValidityPeriod(), [
                'email' => $this->email($request)
            ]);
        } else {
            $token = null;
        }
        return response()->json([
            'token' => $token,
        ]);
    }
}
```

### Broadcasting example
Create event (for example SendMessage) with artisan `php artisan make:event SendMessageEvent`

```php
namespace App\Events;

use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class SendMessageEvent implements ShouldBroadcastNow
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * @var string Message text
     */
    private $messageText;

    public function __construct(string $messageText)
    {
        $this->messageText = $messageText;
    }

    /**
     * The event's broadcast name.
     *
     * @return string
     */
    public function broadcastAs()
    {
        return 'message.new';
    }


    /**
     * Get the data to broadcast.
     *
     * @return array
     */
    public function broadcastWith()
    {
        return ['message' => $this->messageText];
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        // Private channel example. The name of the private channel must be written without the $ prefix
        return new PrivateChannel('private:chat');
        
        // Public chat example
        // return new Channel('public:chat');
    }
}
```

### Method for get token
```js
function getToken(url, ctx) {
    return new Promise(async (resolve, reject) => {
        await fetch(url, {
            method: 'POST',
            headers: new Headers({'Content-Type': 'application/json'}),
            body: JSON.stringify(ctx)
        })
            .then(res => {
                if (!res.ok) {
                    throw new Error(`Unexpected status code ${res.status}`);
                }
                return res.json();
            })
            .then(data => {
                resolve(data.token);
            })
            .catch(err => {
                reject(err);
            });
    });
}
```

### Client connection token
```js
import { Centrifuge } from "centrifuge";

const client = new Centrifuge(
    'ws://localhost:8000/connection/websocket',
    {
        token: 'JWT-GENERATED-ON-BACKEND-SIDE',
        getToken: await function (ctx) {
            return getToken('/centrifuge/client-connection-token', ctx);
        }
    }
);

client.connect();
```

> If initial token is not provided, but `getToken` is specified – then
> SDK should assume that developer wants to use token authentication. In
> this case SDK should attempt to get a connection token before
> establishing an initial connection.

### Channel subscription token
```js
// Public channel
const subPublicChannel = client.newSubscription("public:chat").subscribe();

// Private channel
const subPrivateChannel = client.newSubscription("$private:chat", {
    token: 'JWT-GENERATED-ON-BACKEND-SIDE',
    getToken: await function (ctx) {
        // ctx has channel in the Subscription token case.
        return getToken('/centrifuge/channel-connection-token', ctx);
    },
}).subscribe();
```
> If initial token is not provided, but `getToken` is specified – then
> SDK should assume that developer wants to use token authorization for
> a channel subscription. In this case SDK should attempt to get a
> subscription token before initial subscribe.

### A simple client usage example:

```php
<?php
declare(strict_types = 1);

namespace App\Http\Controllers;

use Opekunov\Centrifugo\Centrifugo;
use Illuminate\Support\Facades\Auth;

class ExampleController
{

    public function example(Centrifugo $centrifugo)
    {
        //or $centrifugo = new Centrifugo();
        
        // Send message into channel
        $centrifugo->publish('news', ['message' => 'Hello world']);

        // Generate connection token
        $token = $centrifugo->generateConnectionToken((string)Auth::id(), 0, [
            'name' => Auth::user()->name,
        ]);

        // Generate private channel token
        $expire = now()->addDay(); //or you can use Unix: $expire = time() + 60 * 60 * 24; 
        $apiSign = $centrifugo->generatePrivateChannelToken((string)Auth::id(), 'channel', $expire, [
            'name' => Auth::user()->name,
        ]);

        //Get a list of currently active channels.
        $centrifugo->channels();

        //Get channel presence information (all clients currently subscribed on this channel).
        $centrifugo->presence('news');

    }
}
```

### Available methods

| Name                                                                                | Description                                                                           |
|-------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| publish(string $channel, array $data)                                               | Send message into channel.                                                            |
| broadcast(array $channels, array $data)                                             | Send message into multiple channel.                                                   |
| publishMany(array $data)                                                            | Send multiple data to multiple channels. $data - array of data arrays [channel, data] |
| presence(string $channel)                                                           | Get channel presence information (all clients currently subscribed on this channel).  |
| presenceStats(string $channel)                                                      | Get channel presence information in short form (number of clients).                   |
| history(string $channel)                                                            | Get channel history information (list of last messages sent into channel).            |
| historyRemove(string $channel)                                                      | Remove channel history information.
| unsubscribe(string $channel, string $user)                                          | Unsubscribe user from channel.                                                        |
| disconnect(string $user_id)                                                         | Disconnect user by it's ID.                                                           |
| channels()                                                                          | Get channels information (list of currently active channels).                         |
| info()                                                                              | Get stats information about running server nodes.                                     |
| generateConnectionToken(string $userId, int $exp, array $info)                      | Generate connection token.                                                            |
| generatePrivateChannelToken(string $userId, string $channel, int $exp, array $info) | Generate private channel token.                                                       |

## License

The MIT License (MIT). Please
see [License File](https://github.com/Velman04/laravel-centrifugo-broadcaster/blob/master/LICENSE) for more information.
