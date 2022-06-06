---
title: Slim4 中间件
summary: Slim4 middleware
date: 2020-02-02
tags: ['php']
---

中间件的作用：在app程序 执行前 或 执行后 要运行的代码，用来操作 Request和Response对象。
比如，在防止 跨站点请求伪造 时就需要在应用运行之前验证请求，这就需要用到中间件。

# 中间件基本要求
中间件必须是可调用的，它接受3个参数：

1. \Psr\Http\Message\ServerRequestInterface - PSR7请求对象
2. \Psr\Http\Message\ResponseInterface - PSR7响应对象
3. callable - 可调用的下一个中间件. 
   中间件必须返回一个实例 \Psr\Http\Message\ResponseInterface. 
   每个中间件应该调用下一个中间件并将其作为参数传递给Request和Response对象. 

# 两种形式的中间件

1. 闭包，如下例：

   ```php
   <?php
   /**
   * Example middleware closure
   *
   * @param  \Psr\Http\Message\ServerRequestInterface $request  PSR7 request
   * @param  \Psr\Http\Message\ResponseInterface      $response PSR7 response
   * @param  callable                                 $next     Next middleware
   *
   * @return \Psr\Http\Message\ResponseInterface
   */
   function ($request, $response, $next) {
      $response->getBody()->write('BEFORE');
      $response = $next($request, $response);  // ***用 $next 调用下一个中间件***
      $response->getBody()->write('AFTER');
   
      return $response;
   };
   ```

2. **Invokable类**，用魔术方法__invoke()实现，如下例：

   ```php
   <?php
   class ExampleMiddleware
   {
       /**
        * Example middleware invokable class
        *
        * @param  \Psr\Http\Message\ServerRequestInterface $request  PSR7 request
        * @param  \Psr\Http\Message\ResponseInterface      $response PSR7 response
        * @param  callable                                 $next     Next middleware
        *
        * @return \Psr\Http\Message\ResponseInterface
        */
       public function __invoke($request, $response, $next)
       {
           $response->getBody()->write('BEFORE');
           $response = $next($request, $response);
           $response->getBody()->write('AFTER');
   
           return $response;
       }
   }
   ```

   使用类中间件，必须用 ->add() 方式载入到 $app 或路由 或Group组路由，如下：

   ```php
   $app->add( new ExampleMiddleware() );
   ```

   

# 怎么使用中间件

## 使用应用程序中间件

应用程序中间件会在所有HTTP请求进入时触发执行，用 add() 方式加载中间件，如下例加载闭包形式中间件：

```php
<?php
$app = new \Slim\App();

$app->add(function ($request, $response, $next) {
	$response->getBody()->write('BEFORE');
	$response = $next($request, $response);
	$response->getBody()->write('AFTER');

	return $response;
});

$app->get('/', function ($request, $response, $args) {
	$response->getBody()->write(' Hello ');

	return $response;
});

$app->run();

```

上面程序将输出 HTTP response 下面内容：  

BEFORE Hello AFTER

## 使用路由中间件

路由中间件会在HTTP request请求符合当前指定URI路由时才触发执行，路由中间件必需在 Slim 应用程序路由方法（如 get()或post()）调用后紧接着就指定加载。
每个路由方法会返回一个 \Slim\Route 实例，可以将该实例视为一个上例的 Slim 应用程序，用上面同样方法用 add() 方式加载中间件。如下例：

```php
<?php
$app = new \Slim\App();

$mw = function ($request, $response, $next) {
    $response->getBody()->write('BEFORE');
    $response = $next($request, $response);
    $response->getBody()->write('AFTER');

    return $response;
};

$app->get('/', function ($request, $response, $args) {
	$response->getBody()->write(' Hello ');

	return $response;
})->add($mw);

$app->run();

```

上面程序同样将输出 HTTP response 下面内容：  

BEFORE Hello AFTER

## 使用路由组中间件

路由组中间件与单路由中间件不同在2个方面：

1. 是在单个路由基础上，有更多条件组合才触发执行的；

2. 不是把 [get()、post()或put()] 等方式视为一个应用实例，而是把 gruop() 路由组实为一个应用实例，用 add() 方法加载中间件。
如下例：

```php
<?php

require_once __DIR__.'/vendor/autoload.php';

$app = new \Slim\App();

$app->get('/', function ($request, $response) {
    return $response->getBody()->write('Hello World');
});

$app->group('/utils', function () use ($app) {
    $app->get('/date', function ($request, $response) {
        return $response->getBody()->write(date('Y-m-d H:i:s'));
    });
    $app->get('/time', function ($request, $response) {
        return $response->getBody()->write(time());
    });
})->add(function ($request, $response, $next) {
    $response->getBody()->write('It is now ');
    $response = $next($request, $response);
    $response->getBody()->write('. Enjoy!');

    return $response;
});

```
当访问 /utils/date 时将输出:  
It is now 2015-07-06 03:11:01. Enjoy!

当访问 /utils/time 时会输出：  
It is now 1436148762. Enjoy!

而访问 / 时则不经过中间件，直接输出:    
Hello World

## 中间件里变量的传递

通过 withAttribute() 和 getAttribute() 来传递中间件变量。
如，设置变量：

```php
$request = $request->withAttribute('foo', 'bar');
```

在路由回调中获取变量值：

```php
$foo = $request->getAttribute('foo');
```

---

转载自[Slim 4 PHP 框架零基础学习笔记－中间件](Slim 4 PHP 框架零基础学习笔记－中间件)，作者[如是博博](https://blog.csdn.net/weixin_42527192)