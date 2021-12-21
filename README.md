# Php CORS
## CORS
Para entender básicamente que son los cors, sugiero que los veamos como un mecanismo que usan los clientes como navegadores o cualquier otra intefaz que realice peticiones http para validar si el origen de la petición es compatible con el destino. Esa compatibilidad se define mediante limitaciones o políticas desde el servidor u origen. Si quieres conocer una definición más completa y tecnica, consulte en: [Control de acceso HTTP (CORS)
](https://developer.mozilla.org/es/docs/Web/HTTP/CORS)


### La siguiente solución es para habilitar CORS en PHP empleando dos funciones:
- enableCors: Dependiendo de tu lógica el llamado de esta función se haría al inicio de la ejecución, su alcance es detectar la petición previa de algunos clientes como lo son los navegadores entre otros.
- cors: Retorna un array con la configuración de las cabeceras y la lista blanca de orígenes que podrán hacer peticiones.

```php
if (!function_exists('enableCors')) {
    function enableCors()
    {
        (!headers_sent()) ? header_remove() : true;
        // Allow from any origin
        if (isset($_SERVER['HTTP_ORIGIN'])) {
            // Decide if the origin in $_SERVER['HTTP_ORIGIN'] is one
            // you want to allow, and if so:
            header("Access-Control-Allow-Origin: {$_SERVER['HTTP_ORIGIN']}");
            header('Access-Control-Allow-Credentials: true');
            header('Access-Control-Max-Age: 86400');    // cache for 1 day
        }

        // Access-Control headers are received during OPTIONS requests
        if ($_SERVER['REQUEST_METHOD'] == 'OPTIONS') {
            header('Access-Control-Allow-Credentials: true');
            header("Allow: GET, POST, OPTIONS, PUT, DELETE, HEAD");
            header("Access-Control-Allow-Methods: " . isset($_SERVER['HTTP_ACCESS_CONTROL_REQUEST_METHOD']) ? $_SERVER['HTTP_ACCESS_CONTROL_REQUEST_METHOD'] : "GET, POST, PUT, OPTIONS, HEAD");
            header("Access-Control-Allow-Headers: " . isset($_SERVER['HTTP_ACCESS_CONTROL_REQUEST_HEADERS']) ? $_SERVER['HTTP_ACCESS_CONTROL_REQUEST_HEADERS'] : "Content-Type,Authorization,X-Api-Key");
            header("Access-Control-Allow-Origin: " . isset($_SERVER['HTTP_ORIGIN']) ? $_SERVER['HTTP_ORIGIN'] : '*');
            http_response_code(200);
            exit(0);
            die;
        }
    }
}

if (!function_exists('cors')) {
    function cors()
    {
        (!headers_sent()) ? header_remove() : true;
        $config = [
            'whitelist'   => ['http://localhost:3000', 'https://mydomain.com', 'http://localhost', '::1'],
            'headers' => [
                "Access-Control-Allow-Credentials" => "true",
                "Access-Control-Allow-Headers" => "X-API-KEY, Origin, X-Requested-With, User-Agent, Content-Type, Accept, Accept-Encoding, Accept-Language,Connection,Host,Origin,Access-Control-Request-Method,Access-Control-Request-Headers,Access-Control-Max-Age,Referer,Authorization",
                "Access-Control-Allow-Methods" => "GET, POST, OPTIONS, PUT, DELETE",
                "Allow" => "GET, POST, OPTIONS, PUT, DELETE, HEAD"
            ]
        ];
        $origin = match (true) {
            (!empty($_SERVER['HTTP_CLIENT_IP'])) => $_SERVER['HTTP_CLIENT_IP'],
            (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) => $_SERVER['HTTP_X_FORWARDED_FOR'],
            default => $_SERVER['REMOTE_ADDR']
        };
        $origin = ($origin === "::1") ? "*" : $origin;
        if (in_array($origin, $config['whitelist']))
            $config['headers']["Access-Control-Allow-Origin"] =  "{$origin}";

        return $config['headers'];
    }
}

```

### Nota: 
Si estas usando Nginx también deberás permitir el método OPTIONS o hacer un pequeño truco para cambiar una respuesta de código http 405 a 200:
- Opción 1: Detectar si el método de la petición es OPTIONS y establecer unas cabeceras y un código de respuesta http.
- Opción 2: Definir un valor para la respuesta del código de error 405. 

### Opción 1

```nginx
location / {
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain; charset=utf-8';
            add_header 'Content-Length' 0;
            return 204;
        }
}

```
### Opción 2

```nginx
location / {
        error_page 405 =200 $uri;
}

```
