# Setting up your free SSL for Testing

- This is a practical example of how to set up your own self signed SSL and use it for your Laravel Websockets.
- [Laravel](https://laravel.com/) version : ^10.10
- PHP version : ^8.1
- Apache: ^2.4.51
- [BeyondCode Laravel Websockets](https://beyondco.de/docs/laravel-websockets/getting-started/introduction): ^1.14

## WAMP

### Generating SSL

You will need to generate a free SSL from your wamp openssl.

1. Create a folder C:\SSL

2. Generate SSL certificate. You need to open a command prompt and **"Run as Administrator"**. Go to your SSL folder directory.
```shell
cd C:\SSL
```

3. Replace apache2.4.51 of your wamp's apache version. You will need to enter passphrase.
```shell
C:\wamp64\bin\apache\apache2.4.51\bin\openssl genrsa -aes256 -out private.key 2048
```

4. Replace apache2.4.51 of your wamp's apache version. You will need to enter passphrase.
```shell
C:\wamp64\bin\apache\apache2.4.51\bin\openssl rsa -in private.key -out private.key
```

5. Replace apache2.4.51 of your wamp's apache version.. Answer the inputs. Make sure to enter FQDN as 127.0.0.1 or localhost.
```shell
C:\wamp64\bin\apache\apache2.4.51\bin\openssl req -new -x509 -nodes -sha1 -key private.key -out certificate.crt -days 36500 -config c:\wamp64\bin\apache\apache2.4.51\conf\openssl.cnf
```

#### Output:
```shell
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:PH
State or Province Name (full name) [Some-State]:SomewhereState
Locality Name (eg, city) []:SomewhereCity
Organization Name (eg, company) [Internet Widgits Pty Ltd]:SomeCompany
Organizational Unit Name (eg, section) []:SomeOrganization
Common Name (e.g. server FQDN or YOUR name) []:localhost
Email Address []:support@example.com
```

6. Open C:\wamp64\bin\apache\apache2.4.51\conf\httpd.conf. Make sure to uncomment the modules below:
```
LoadModule ssl_module modules/mod_ssl.so
Include conf/extra/httpd-ssl.conf
LoadModule socache_shmcb_module modules/mod_socache_shmcb.so
```

### Setting up your wamp http-vhost

You will need to properly configure your wamp to handle https / ssl.

1. Create a folder directory `C:\wamp64\bin\apache\apache2.4.51\conf\key`
2. Copy all the files from `C:\SSL` to `C:\wamp64\bin\apache\apache2.4.51\conf\key`
3. Open the file `C:\wamp64\bin\apache\apache2.4.51\conf\extra\httpd-ssl.conf`
4. Update the inputs below:

```
DocumentRoot "C:/wamp64/www"
ServerName localhost:443
ServerAdmin admin@example.com
SSLCertificateFile "${SRVROOT}/conf/key/certificate.crt"
SSLCertificateKeyFile "${SRVROOT}/conf/key/private.key"
```

### Setup your Laravel Application

### Install Laravel Websockets (Skip installation if already existing)

1. Install

```shell
composer require beyondcode/laravel-websockets
```
```shell
php artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="migrations"
```
```shell
php artisan migrate
```
```shell
php artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="config"
```

2. Configure your laravel websocket. Open `config/websockets.php`

```php
'ssl' => [
    /*
        * Path to local certificate file on filesystem. It must be a PEM encoded file which
        * contains your certificate and private key. It can optionally contain the
        * certificate chain of issuers. The private key also may be contained
        * in a separate file specified by local_pk.
        */
    'local_cert' => env('LARAVEL_WEBSOCKETS_SSL_LOCAL_CERT', null),

    /*
        * Path to local private key file on filesystem in case of separate files for
        * certificate (local_cert) and private key.
        */
    'local_pk' => env('LARAVEL_WEBSOCKETS_SSL_LOCAL_PK', null),

    /*
        * Passphrase for your local_cert file.
        */
    'passphrase' => env('LARAVEL_WEBSOCKETS_SSL_PASSPHRASE', null),
    'verify_peer' => false,
],
```

3. Update your `config/broadcasting.php`

```php
'pusher' => [
    'driver' => 'pusher',
    'key' => env('PUSHER_APP_KEY'),
    'secret' => env('PUSHER_APP_SECRET'),
    'app_id' => env('PUSHER_APP_ID'),
    'options' => [
        'cluster' => env('PUSHER_APP_CLUSTER'),
        'host' => env('PUSHER_HOST') ?: 'api-' . env('PUSHER_APP_CLUSTER', 'mt1') . '.pusher.com',
        'port' => env('PUSHER_PORT', 6001),
        'scheme' => env('PUSHER_SCHEME', 'https'),
        'encrypted' => true,
        'useTLS' => env('PUSHER_SCHEME', 'https') === 'https',
    ],
    'client_options' => [
        // Guzzle client options: https://docs.guzzlephp.org/en/stable/request-options.html
        'verify' => false
    ],
    // for self-signed ssl
    'curl_options' => [
        CURLOPT_SSL_VERIFYHOST => 0,
        CURLOPT_SSL_VERIFYPEER => 0,
    ],
],
```

4. Uncomment the laravel echo from `resources/js/bootstrap.js`. Update your configurations from the config below.

```js
import Echo from 'laravel-echo';

import Pusher from 'pusher-js';
window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER ?? 'mt1',
    wsHost: import.meta.env.VITE_PUSHER_HOST ? import.meta.env.VITE_PUSHER_HOST : `ws-${import.meta.env.VITE_PUSHER_APP_CLUSTER}.pusher.com`,
    //wsPort: import.meta.env.VITE_PUSHER_PORT ?? 80,
    //wssPort: import.meta.env.VITE_PUSHER_PORT ?? 443,
    wsPort: 6001,
    wssPort: 6001,
    forceTLS: (import.meta.env.VITE_PUSHER_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});
```

In .env, update your broadcast driver and add the self signed SSL. Make sure you replace your apache version apache2.4.51.
```
BROADCAST_DRIVER=pusher

PUSHER_APP_ID=<your-pusher-id>
PUSHER_APP_KEY=<your-pusher-key>
PUSHER_APP_SECRET=<your-pusher-secret>
PUSHER_HOST=localhost
PUSHER_PORT=6001
PUSHER_SCHEME=https
PUSHER_APP_CLUSTER=ap1

LARAVEL_WEBSOCKETS_SSL_LOCAL_CERT=C:\wamp64\bin\apache\apache2.4.51\conf\key\certificate.crt
LARAVEL_WEBSOCKETS_SSL_LOCAL_PK=C:\wamp64\bin\apache\apache2.4.51\conf\key\private.key
LARAVEL_WEBSOCKETS_SSL_PASSPHRASE=<your-passphrase>
```

### Set up the PHP cacert for curl

You will need to setup the cacert certificate validation for your curl request in PHP.

1. Download the latest cacert.pem [here](https://curl.se/docs/caextract.html)

2. Copy the cacert.pem file to `C:\wamp64\bin\apache\apache2.4.51\conf\key\cacert.pem`

3. Edit your php.ini.

Replace paths to your cacert.pem and cerificate.crt
```
curl.cainfo = C:\wamp64\bin\apache\apache2.4.51\conf\key\cacert.pem
```
```
openssl.cafile=C:\wamp64\bin\apache\apache2.4.51\conf\key\certificate.crt
```

4. Save all your changes.

5. **Finally, restart your WAMP SERVER.**

6. Also restart your **laravel websockets**

### Testing

1. Run your websocket app dashboard from wamp localhost. https://localhost/PHP-PracticalExample-LaravelWebsockets/public/laravel-websockets

2. Try to send an event.

## LARAGON
- TO BE TESTED.

## ORACLE LINUX APACHE
- TO BE TESTED.

## ACTUAL ORACLE LINUX SSL
- TO BE TESTED.

