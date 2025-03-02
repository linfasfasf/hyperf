# EasyWechat

[EasyWeChat](https://www.easywechat.com/) 是一个开源的微信 SDK (非微信官方 SDK)。

> 因为组件默认使用 `Curl`，所以我们需要修改对应的 `GuzzleClient` 为协程客户端，或者修改常量 `SWOOLE_HOOK_FLAGS` 为 `SWOOLE_HOOK_ALL | SWOOLE_HOOK_CURL`

## 替换 `Handler`

以下以小程序为例，

```php
<?php

use Hyperf\Utils\ApplicationContext;
use EasyWeChat\Factory;
use EasyWeChat\Kernel\ServiceContainer;
use GuzzleHttp\Client;
use GuzzleHttp\HandlerStack;
use Hyperf\Guzzle\CoroutineHandler;
use Hyperf\Guzzle\HandlerStackFactory;
use Overtrue\Socialite\Providers\AbstractProvider;
use Symfony\Component\HttpFoundation\ParameterBag;
use Symfony\Component\HttpFoundation\RedirectResponse;

$container = ApplicationContext::getContainer();

$app = Factory::miniProgram($config);

// 设置 HttpClient，当前设置没有实际效果，在数据请求时会被 guzzle_handler 覆盖，但不保证 EasyWeChat 后面会修改这里。
$config = $app['config']->get('http', []);
$config['handler'] = $container->get(HandlerStackFactory::class)->create();
$app->rebind('http_client', new Client($config));

// 重写 Handler
$app['guzzle_handler'] = new CoroutineHandler();

// 设置 OAuth 授权的 Guzzle 配置
AbstractProvider::setGuzzleOptions([
    'http_errors' => false,
    'handler' => HandlerStack::create(new CoroutineHandler()),
]);
```

## 修改 `SWOOLE_HOOK_FLAGS`

修改入口文件 `bin/hyperf.php`，以下忽略不需要修改的代码。

```php
<?php

! defined('SWOOLE_HOOK_FLAGS') && define('SWOOLE_HOOK_FLAGS', SWOOLE_HOOK_ALL | SWOOLE_HOOK_CURL);
```

## 如何使用 EasyWeChat

`EasyWeChat` 是为 `PHP-FPM` 架构设计的，所以在某些地方需要修改下才能在 Hyperf 下使用。下面我们以支付回调为例进行讲解。

1. `EasyWeChat` 中自带了 `XML` 解析，所以我们获取到原始 `XML` 即可。

```php
$xml = $this->request->getBody()->getContents();
```

2. 将 XML 数据放到 `EasyWeChat` 的 `Request` 中。

```php
<?php
use Symfony\Component\HttpFoundation\Request;

$get = $this->request->getQueryParams();
$post = $this->request->getParsedBody();
$cookie = $this->request->getCookieParams();
$files = $this->request->getUploadedFiles();
$server = $this->request->getServerParams();
$xml = $this->request->getBody()->getContents();

$app['request'] = new Request($get,$post,[],$cookie,$files,$server,$xml);

// Do something...

```

3. 服务器配置

如果需要使用微信公众平台的服务器配置功能，可以使用以下代码。

> 以下 `$response` 为 `Symfony\Component\HttpFoundation\Response` 并非 `Hyperf\HttpMessage\Server\Response` 
> 所以只需将 `Body` 内容直接返回，即可通过微信验证。

```php
$response = $app->server->serve();

return $response->getBody()->getContents();
```

## 如何替换缓存

`EasyWeChat` 默认使用 `文件缓存`，而现实场景是 `Redis` 缓存居多，所以这里可以替换成 `Hyperf` 提供的 `hyperf/cache` 缓存组件，如您当前没有安装该组件，请执行 `composer require hyperf/cache` 引入，使用示例如下：

```php
<?php
use Psr\SimpleCache\CacheInterface;
use Hyperf\Utils\ApplicationContext;
use EasyWeChat\Factory;

$app = Factory::miniProgram([]);
$app['cache'] = ApplicationContext::getContainer()->get(CacheInterface::class);
```
