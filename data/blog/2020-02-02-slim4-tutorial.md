---
title: Slim4教程 tutorial
summary: php REST API by Slim4 - This tutorial shows you how to work with the powerful and lightweight Slim 4 framework.
date: 2020-02-02
tags: ['php', 'REST API']
---

This tutorial shows you how to work with the powerful and lightweight Slim 4 framework.

这篇教程将向你展示如何使用轻量级且功能强大的Slim4框架。

Slim Framework is a great microframework for web applications, RESTful API's and websites.

Our aim is to create a RESTful API with routing, business logic and database operations.

Standards like [PSR](https://www.php-fig.org/psr/) and best practices are very important and integrated part of this tutorial.

Slim框架是一款面向web app，REST API以及网站的轻量级框架。我们的目标是编写一个具备路由，业务逻辑以及数据库操作的RESTful API。在这篇教程中，一些标准（例如PSR）将非常重要。

## Requirements 准备

* PHP 7.1+
* MySQL 5.7+
* Apache webserver
* Composer

## Installation 安装

Create a new project directory and run this command to install the Slim 4 core components:

创建一个新工程文件夹，在此文件夹下运行如下指令以安装Slim4核心组件：

```
composer require slim/slim:"4.*"
```

In Slim 4 the PSR-7 implementation is decoupled from the App core. 
This means you can also install other PSR-7 implementations like [nyholm/psr7](https://github.com/Nyholm/psr7).

在Slim4框架中，PSR-7被从核心组件中剥离开，这意味着你可以安装其他类型的的PSR-7，例如[nyholm/psr7](https://github.com/Nyholm/psr7).

In our case we are installing the Slim PSR-7 implementations using this command:

在我们的教程中，我们将安装由Slim实现的PSR-7。运行如下指令：

```
composer require slim/psr7
```

Now we install a number of useful convenience methods such as `$response->withJson()`:

现在我们要安装一系列非常有用的工具方法，例如`$response->withJson()`:

```
composer require slim/http
```

As next we need a PSR-11 container implementation for **dependency injection** and **autowiring**.

下一步我们需要一个PSR-11 container容器，以便于**依赖注入**以及**自动关联**。

Run this command to install [PHP-DI](http://php-di.org/):

运行如下指令安装[PHP-DI](http://php-di.org/): （中文文档：[php-di中文文档](https://www.kancloud.cn/shanyu/php-di/182580)）

```
composer require php-di/php-di
```

To access the application configuration install the `selective/config` package:

安装`selective/config`以对app做配置：

```
composer require selective/config
```

For testing purpose we are installing [phpunit](https://phpunit.de/) as development dependency with the `--dev` option:

安装 [phpunit](https://phpunit.de/)来做测试：

```
composer require phpunit/phpunit --dev
```

Ok nice, now we have installed the most basic dependencies for our project. Later we will add more.

我们现在已经基本安装好了依赖，之后可能会加入其它依赖。

**Note:** Please don't commit the `vendor/` to your git repository. To set up the git repository correctly, create a file called `.gitignore` in the project root folder and add the following lines to this file:

请不要将`vendor/`加入git仓库，可以通过编写`.gitignore`文件来设置git。

```
vendor/
.idea/
```

## Directory structure 文件结构

A good directory structure helps you organize your code, simplifies setup on the webserver and increases the security of the entire application.

一个好的文件夹结构能帮助你更好地理清代码间的关系，简化服务器配置以及提高app安全性。

Create the following directory structure in the root directory of your project:

新建如下图的文件夹。

```
.
├── config/             Configuration files
├── public/             Web server files (DocumentRoot)
│   └── .htaccess       Apache redirect rules for the front controller
│   └── index.php       The front controller
├── templates/          Twig templates
├── src/                PHP source code (The App namespace)
├── tmp/                Temporary files (cache and logfiles)
├── vendor/             Reserved for composer
├── .htaccess           Internal redirect to the public/ directory
└── .gitignore          Git ignore rules
```

In a web application, it is important to distinguish between the public and 
non-public areas.

在一个web app中，需要分清公共区域和非公共区域之间的关系。

The `public/` directory serves your application and will therefore also be 
directly accessible by all browsers, search engines and API clients. 
All other folders are not public and must not be accessible online. 
This can be done by defining the `public` folder in Apache as `DocumentRoot` 
of your website. But more about that later.

`public/`文件夹用来服务于你的app因此它也必须是对所有浏览器，搜索引擎以及API客户端**可见**。其它非公共文件夹在线上应当为**不可见**。可以通过设置apache服务器的public文件夹为document root。


## Apache URL rewriting 重写apache url

To run a Slim app with apache we have to add url rewrite rules to redirect the web traffic to a so called [front controller](https://en.wikipedia.org/wiki/Front_controller).

为了在apache下启动我们的Slim app，我们需要定义重定向规则，也就是一个前端控制器。

The front controller is just a `index.php` file and the entry point to the application.

前段控制器其实就是一个`index.php`文件，它是我们的app的入口。

* Create a directory: `public/` 新建`public/`文件夹（在上一部分应当已经建好）

* Create a `.htaccess` file in your `public/` directory and copy/paste this content:

  在改文件夹下新建`.htaccess`文件，拷贝如下内容：

```htaccess
# Redirect to front controller
RewriteEngine On
# RewriteBase /
RewriteCond %{REQUEST_FILENAME} !-d
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule ^ index.php [QSA,L]
```

Please **don't** change the `RewriteRule` directive. It must be exactly as shown above.

请不要修改`RewriteRule`规则，它必须和上面代码一致。

* Create a second `.htaccess` file in your project root-directory and copy/paste this content:

  在app文件夹根目录下新建第二个`.htaccess`文件，拷贝以下内容：

```htaccess
RewriteEngine on
RewriteRule ^$ public/ [L]
RewriteRule (.*) public/$1 [L]
```

Don't skip this step. This second `.htaccess` file is important to run your Slim app in a sub-directory
and within your development environment. 

不要跳过这一步，如果你的Slim app在服务器的子文件夹下运行，那么第二个`.htaccess`文件对你的Slim app至关重要。

* Create the front-controller file `public/index.php` and copy/paste this content:

  新建前端控制器文件`public/index.php`，拷贝以下内容

```php
<?php

(require __DIR__ . '/../config/bootstrap.php')->run();
```

The [front controller](https://en.wikipedia.org/wiki/Front_controller) is the entry point 
to your slim application and handles all requests by channeling requests through a single handler object.

前端控制器是你的Slim app的入口，它通过一个处理器对象处理所有请求。

## Configuration 配置

The directory for all configuration files is: `config/`

将配置文件放在`config/`目录下。

The file `config/settings.php` is the main configuration file and combines 
the default settings with environment specific settings. 

`config/settings.php`是生产环境的配置文件。

* Create a directory: `config/`

  新建目录`config/`

* Create a configuration file `config/settings.php` and copy/paste this content:

  新建配置文件`config/settings.php` ，拷贝以下内容

```php
<?php

// Error reporting
error_reporting(0);
ini_set('display_errors', '0');

// Timezone
date_default_timezone_set('Europe/Berlin');

// Settings
$settings = [];

// Path settings
$settings['root'] = dirname(__DIR__);
$settings['temp'] = $settings['root'] . '/tmp';
$settings['public'] = $settings['root'] . '/public';

// Error Handling Middleware settings
$settings['error_handler_middleware'] = [

    // Should be set to false in production
    'display_error_details' => true,

    // Parameter is passed to the default ErrorHandler
    // View in rendered output by enabling the "displayErrorDetails" setting.
    // For the console and unit tests we also disable it
    'log_errors' => true,

    // Display error details in error log
    'log_error_details' => true,
];

return $settings;
```

### Startup 启动

The app startup process contains the code that is executed when the application (request) is started. 

当收到数据请求时，app便会启动

The bootstrap procedure includes the composer autoloader and then continues to
build the container, creates the app and registers the routes + middleware entries.

启动引导过程注册各个类库的类文件自动加载器，之后建立容器，实例化app以及注册各个路由和中间件入口。

Create the bootstrap file `config/bootstrap.php` and copy/paste this content:

新建`config/bootstrap.php`文件，拷贝以下内容：

```php
<?php

use DI\ContainerBuilder;
use Slim\App;

require_once __DIR__ . '/../vendor/autoload.php';

$containerBuilder = new ContainerBuilder();

// Set up settings
$containerBuilder->addDefinitions(__DIR__ . '/container.php');

// Build PHP-DI Container instance
$container = $containerBuilder->build();

// Create App instance
$app = $container->get(App::class);

// Register routes
(require __DIR__ . '/routes.php')($app);

// Register middleware
(require __DIR__ . '/middleware.php')($app);

return $app;
```

### Routing setup 设置路由

Create a file for all routes `config/routes.php` and copy/paste this content:

新建一个文件`config/routes.php`用于配置所有路由，拷贝以下代码，目前暂时是空的。

```php
<?php

use Slim\App;

return function (App $app) {
    // empty
};

```

## Middleware 中间件

### What is a middleware? 什么是中间件

A middleware can be executed before and after your Slim application to manipulate the request and response object according to your requirements.

一个中间件可以根据你的需求，在Slim app启动前及启动后运行。

[Read more](http://www.slimframework.com/docs/v4/concepts/middleware.html)

### Routing and error middleware 路由及错误中间件

Create a file to load global middleware handler `config/middleware.php` and copy/paste this content:

新建文件`config/middleware.php`，用于加载全局中间件：（JSON，路由，错误处理）

```php
<?php

use Selective\Config\Configuration;
use Slim\App;

return function (App $app) {
    // Parse json, form data and xml
    $app->addBodyParsingMiddleware();

    // Add routing middleware
    $app->addRoutingMiddleware();

    $container = $app->getContainer();
    
    // Add error handler middleware
    $settings = $container->get(Configuration::class)->getArray('error_handler_middleware');
    $displayErrorDetails = (bool)$settings['display_error_details'];
    $logErrors = (bool)$settings['log_errors'];
    $logErrorDetails = (bool)$settings['log_error_details'];

    $app->addErrorMiddleware($displayErrorDetails, $logErrors, $logErrorDetails);
};

```

## Container 容器

### A quick guide to the container 容器快速入门

*（本段翻译不准确）*

**[Dependency injection](https://en.wikipedia.org/wiki/Dependency_injection)** is passing dependency to other objects.
Dependency injection makes testing easier. The injection can be done through a constructor.

依赖注入，即将依赖给予其它对象，它使得测试更简单。通过构造函数即可完成注入。

A **dependencies injection container** (DIC) is a tool for injecting dependencies.

一个**依赖注入容器**是一个注入依赖的工具。

**A general rule:** The core application should not use the container.
Injecting the container into a class is an **anti-pattern**. You should declare all class 
dependencies in the constructor explicitly. 

app核心部分不应该使用容器。将一个依赖注入容器注入到类中是**反设计模式**，（隐式地建立了对于容器的依赖，而不是真正需要替换的依赖，而且还会让你的代码更不透明，最终变得更难测试）。你应该在构造函数中显式地声明所有依赖类。

Why is injecting the container (in most cases) an anti-pattern?

为什么把容器注入到类中是一种反设计模式（大多数情况下）？

In Slim 3 the [Service Locator](https://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/) (anti-pattern) was the
default "style" to inject the whole ([Pimple](https://pimple.symfony.com/)) container and fetch the dependencies from it. 
However, there are the following disadvantages:

在Slim3中，服务定位模式（反设计模式）是一种默认的风格：将整个容器注入进去然后从中获取依赖。然而，这种模式有以下几个缺点：

* The Service Locator (anti-pattern) **hides the actual dependencies** of your class. 

  服务定位器隐藏了你的类中所需要的依赖

* The Service Locator (anti-pattern) also violates the [Inversion of Control](https://en.wikipedia.org/wiki/Inversion_of_control) (IoC) principle of [SOLID](https://en.wikipedia.org/wiki/SOLID).

  服务定位器也违反了SOLID规则中的IoC反转控制。

Q: How can I make it better?  我该怎样做？

A: Use [composition over inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance) 
and (constructor) **dependency injection**. 

使用组合大于依赖以及（构造函数）以及**依赖注入**。

Dependency injection is a programming practice of passing into an object it’s collaborators, 
rather the object itself creating them. 

依赖注入是一种良好的设计模式，将参数赋予对象的构造函数，而不是要求对象本身实例化。

Since **Slim 4** you can use modern tools like [PHP-DI](http://php-di.org/) with the awesome [autowire](http://php-di.org/doc/autowiring.html) feature. 
This means: Now you can declare all dependencies explicitly in your constructor and let the DIC inject these 
dependencies for you. 

自Slim4开始，你可以使用像php-di一类具有**自动装配**功能的工具，这也就是说：现在你可以在构造函数内显式地声明所需要的依赖，让DIC来替你注入这些依赖。

To be more clear: Composition has nothing to do with the "autowire" feature of the DIC. You can use composition 
with pure classes and without a container or anything else. The autowire feature just uses the 
[PHP Reflection](https://www.php.net/manual/en/book.reflection.php) classes to resolve and inject the 
dependencies automatically for you.

“组合”与DIC的“自动装配”没有任何联系，你可以单单使用类来组合而不需要容器或者其他的组件。自动装配通过使用php反射来帮助你自动地解决病注入依赖。

### Container definitions 容器的定义

Slim 4 uses a dependency injection container to prepare, manage and inject application dependencies. 

Slim4使用依赖注入容器来准备，管理以及注入app所需要的依赖。

You can add any container library that implements the [PSR-11](https://www.php-fig.org/psr/psr-11/) interface.

你可以使用任何容器库，只要他们实现了[PSR-11](https://www.php-fig.org/psr/psr-11/)接口就行。

Create a new file for the container entries `config/container.php` and copy/paste this content:

新建`config/container.php`文件，拷贝以下内容：

```php
<?php

use Psr\Container\ContainerInterface;
use Selective\Config\Configuration;
use Slim\App;
use Slim\Factory\AppFactory;

return [
    Configuration::class => function () {
        return new Configuration(require __DIR__ . '/settings.php');
    },

    App::class => function (ContainerInterface $container) {
        AppFactory::setContainer($container);
        $app = AppFactory::create();

        // Optional: Set the base path to run the app in a sub-directory
        // The public directory must not be part of the base path
        //$app->setBasePath('/slim4-tutorial');

        return $app;
    },

];
```

## Base path 路径

If you run your Slim app in a sub-directory, resp. not directly within the 
[DocumentRoot](https://httpd.apache.org/docs/2.4/en/mod/core.html#documentroot)
of your webserver, you must set the "correct" base path.

如果你不是直接在[DocumentRoot](https://httpd.apache.org/docs/2.4/en/mod/core.html#documentroot)下运行你的app，那么请务必设置好启动路径

Ideally the `DoumentRoot` of your production server points directly to the `public/` directory.

In all other cases you have to make sure, that your base path is correct. For example,
the DocumentRoot directory is `/var/www/domain.com/htdocs/`, but the application
is stored under `/var/www/domain.com/htdocs/my-app/`, then you have to set `/my-app` as base path.

理想状况：你的服务器仅仅用于这一个Slim app，那么`DoumentRoot` 可以直接指向`public/` 目录（其实也就是这个工程的根目录就是`DoumentRoot`）。如果不是，你可以在`config/container.php`文件中设置一下：（上面的代码中这个设置被注释掉了）

Example:

```php
$app->setBasePath('/my-app');
```

Be careful: The `public/` directory is only the `DoumentRoot` of your webserver, 
but it's never part of your base path and the official url.

注意：不要让`public/`出现在你的url中，他只是作用在你的网络服务器里。

Bad urls 不好的网址
* `http://www.example.com/public`
* `http://www.example.com/public/users`
* `http://www.example.com/my-app/public`
* `http://www.example.com/my-app/public/users`

Good urls: 好的网址
* `http://www.example.com`
* `http://www.example.com/users`
* `http://www.example.com/my-app`
* `http://www.example.com/my-app/users`

## Your first route 你的第一个路由

Open the file `config/routes.php` and insert the code for the first route:

打开`config/routes.php`，插入以下代码，设置你的第一个路由：

```php
<?php

use Slim\Http\Response;
use Slim\Http\ServerRequest;
use Slim\App;

return function (App $app) {
    $app->get('/', function (ServerRequest $request, Response $response) {
        $response->getBody()->write('Hello, World!');

        return $response;
    });
};

```

Now open your website, e.g. http://localhost and you should see the message `Hello, World!`.

打开 http://localhost，你应该可以看到`Hello, World!`。

If you get a **404 error (not found)**, you should define the correct **basePath** in `config/container.php`.

如果你遇到了**404 error (not found)**错误，应该在`config/container.php`设置根目录，例如

**Example:**
```
$app->setBasePath('/slim4-tutorial');
```

## PSR-4 autoloading PSR-4自动加载

For the next steps we have to register the `\App` namespace for the PSR-4 autoloader.

下一步我们需要为PSR-4自动加载器设置`\App`命名空间

Add this autoloading settings into `composer.json`:

将如下设置插入在`composer.json`文件内

```json
"autoload": {
    "psr-4": {
        "App\\": "src"
    }
},
"autoload-dev": {
    "psr-4": {
        "App\\Test\\": "tests"
    }
}
```

The complete `composer.json` file should look like this:

完整的`composer.json`文件应当看起来像如下代码：

```json
{
    "require": {
        "php-di/php-di": "^6.0",
        "selective/config": "^0.1.1",
        "slim/http": "^1",
        "slim/psr7": "^1",
        "slim/slim": "^4.4"
    },
    "require-dev": {
        "phpunit/phpunit": "^8.4"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "App\\Test\\": "tests"
        }
    },
    "config": {
        "process-timeout": 0,
        "sort-packages": true
    }
}
```

Run `composer update` for the changes to take effect.

运行`composer update`以生效。

## Action 动作

Each **Single Action Controller** is represented by a individual class or closure.

类或闭包函数代表了每个**单独的动作控制器**。

The *Action* does only these things:

*Action*仅做如下事情：

* collects input from the HTTP request (if needed)

  从HTTP请求中收集输入参数

* invokes the **Domain** with those inputs (if required) and retains the result

  用这些参数（如果需要的话）调用**域名**并保留结果

* builds an HTTP response (typically with the Domain invocation results).

  创建一个HTTP回复（使用API域名所产生的数据报）

All other logic, including all forms of input validation, error handling, and so on, 
are therefore pushed out of the Action and into the **Domain** 
(for domain logic concerns) or the response renderer (for presentation logic concerns). 

对所有的其它业务逻辑，包含了输入参数，错误处理等其它模块，都会被注入进Action以及域名以及回复处理器。

A response could be rendered to HTML (e.g with Twig) for a standard web request; or 
it might be something like JSON for RESTful API requests.

对于一个标准的web请求，回复可能被处理为HTML格式，或者对于RESTful API请求来说，回复应当是JSON格式。

**Note:** [Closures](https://www.php.net/manual/en/class.closure.php) (functions) as routing 
handlers are quite "expensive", because PHP has to create all closures for each request. 
The use of class names is more lightweight, faster and scales better for larger applications.

**注意**：用闭包函数作为路由处理的代价是相当昂贵的，因为php将对每个请求都建立一个闭包函数。对大型app，使用类名是更加轻量，快速的。

More details about the flow of everything that happens when arriving a route 
and the communication between the different layers can be found here: [Action](https://odan.github.io/slim4-skeleton/action.html)

阅读更多有关于路由及各个层信息传输的教程： [Action](https://odan.github.io/slim4-skeleton/action.html)

* Create a directory: `src/`

  新建文件夹`src/`

* Create a sub-directory: `src/Action`

  新建子文件夹：`src/Action`

* Create this action class in: `src/Action/HomeAction.php`

  新建文件`src/Action/HomeAction.php`，插入这个action类

```php
<?php

namespace App\Action;

use Slim\Http\Response;
use Slim\Http\ServerRequest;

final class HomeAction
{
    public function __invoke(ServerRequest $request, Response $response): Response
    {
        $response->getBody()->write('Hello, World!');

        return $response;
    }
}
```

Then open `config/routes.php` and replace the route closure for `/` with this line:

打开`config/routes.php` ，将闭包路由替换为如下：

```php
$app->get('/', \App\Action\HomeAction::class);
```

The complete `config/routes.php` should look like this now:

现在完整的`config/routes.php`应该看起来如下：

```php
<?php

use Slim\App;

return function (App $app) {
    $app->get('/', \App\Action\HomeAction::class);
};
```

Now open your website, e.g. http://localhost and you should see the message `Hello, World!`.

打开 http://localhost，你应该可以看到`Hello, World!`。

### Writing JSON to the response 将JSON写入回复中

Instead of calling `json_encode` everytime, you can use the `withJson()` method to render the response.

你可以使用`withJson()`方法来生成回复，而不是每次都使用`json_encode`方法。

```php
<?php

namespace App\Action;

use Slim\Http\Response;
use Slim\Http\ServerRequest;

final class HomeAction
{
    public function __invoke(ServerRequest $request, Response $response): Response
    {
        return $response->withJson(['success' => true]);
    }
}
```

Open your website, e.g. http://localhost and you should see the JSON response `{"success":true}`.

打开 http://localhost，你应该可以看到JSON回复`{"success":true}`。

To change to http status code, just use the `$response->withStatus(x)` method. Example:

如若需更改http状态码，使用`$response->withStatus(x)`方法。

```php
$result = ['error' => ['message' => 'Validation failed']];
        
return $response->withJson($result)->withStatus(422);
```

## Domain 域名

Forget CRUD! Your API should reflect the business [use cases](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) 
and not the technical "database operations" aka. [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete). 
Don't put business logic into actions. The action invokes the domain layer, 
resp. the application service. If you want to reuse the same logic in another action, 
then just invoke that application service you need in your action.

忘记CRUD！你的API应该体现**业务用例**，而不是什么技术方面的所谓「数据库操作」CRUD。不要把业务逻辑放进action里，action仅仅是调用域名层，即service。如果你想在其他action内复用相同的业务逻辑，那就在action中去调用你所需要的service。

### Services 服务

The [Domain](https://github.com/pmjones/adr/blob/master/ADR.md#model-versus-domain) is the place for the
complex [business logic](https://en.wikipedia.org/wiki/Business_logic).

域名是你放置复杂业务逻辑的地方。

Instead of putting the logic into gigantic (fat) "Models", we put the logic into smaller, 
specialized **Service** classes, aka Application Service.

我们将逻辑分为更小的Service类，又名Application Service，而不是将复杂的逻辑放在Model层。

A service provides a specific functionality or a set of functionalities, such as the retrieval of 
specified information or the execution of a set of operations, with a purpose that different clients 
can reuse for different purposes.

一个service提供了一整套详细的功能，例如搜集一些详细的信息或执行一系列操作，每个不同的客户都可以以不同的目的复用service。

There can be multiple clients for a service, e.g. the action (request), 
another service, the CLI (console) and the unit-test environment (phpunit).

一个service可以有很多客户：action（请求），其他service，CLI（console）以及单元测试。

> A service class is not a "Manager" or "Utility" class.

Each service class should have only one responsibility, e.g. to transfer money from A to B, and not more.

每个service都应该有且只有一个责任，比如说从A转账到B。

Separate **data** from **behavior** by using services for the behavior and DTO's for the data.

用service和DTO将**数据**与**行为**分离

The directory for all (domain) modules and sub-modules is: `src/Domain`

`src/Domain`是存放域名模块的文件，伪代码如下：

**Pseudo example:**

```php
use App\Domain\User\Data\UserCreateData;
use App\Domain\User\Service\UserCreator;

$user = new UserCreateData();
$user->username = 'john.doe';
$user->firstName = 'John';
$user->lastName = 'Doe';
$user->email = 'john.doe@example.com';

$service = new UserCreator();
$service->createUser($user);
```

### Data Transfer Objects (DTO) 数据转移对象DTO

A DTO contains only pure **data**. There is no business or domain specific logic. 
There is also no database access within a DTO. 

一个DTO只负责处理数据，没有具体的业务逻辑，也没有数据库操作。

A service fetches data from a repository and  the repository (or the service) 
fills the DTO with data. A DTO can be used to transfer data inside or outside the domain.

一个service从仓库中获取数据，仓库（或service）用数据填充DTO。一个DTO可以在域名内外用来处理数据。

Create a DTO class to hold the data in this file: `src/Domain/User/Data/UserCreateData.php`

新建一个DTO类`src/Domain/User/Data/UserCreateData.php`，用来保存数据。

```php
<?php

namespace App\Domain\User\Data;

final class UserCreateData
{
    /** @var string */
    public $username;

    /** @var string */
    public $firstName;

    /** @var string */
    public $lastName;

    /** @var string */
    public $email;
}
```

Create the code for the service class `src/Domain/User/Service/UserCreator.php`:

为`src/Domain/User/Service/UserCreator.php`赋予代码：

```php
<?php

namespace App\Domain\User\Service;

use App\Domain\User\Data\UserCreateData;
use App\Domain\User\Repository\UserCreatorRepository;
use UnexpectedValueException;

/**
 * Service.
 */
final class UserCreator
{
    /**
     * @var UserCreatorRepository
     */
    private $repository;

    /**
     * The constructor.
     *
     * @param UserCreatorRepository $repository The repository
     */
    public function __construct(UserCreatorRepository $repository)
    {
        $this->repository = $repository;
    }

    /**
     * Create a new user.
     *
     * @param UserCreateData $user The user data
     *
     * @return int The new user ID
     */
    public function createUser(UserCreateData $user): int
    {
        // Validation
        if (empty($user->username)) {
            throw new UnexpectedValueException('Username required');
        }

        // Insert user
        $userId = $this->repository->insertUser($user);

        // Logging here: User created successfully

        return $userId;
    }
}
```

Take a look at the **constructor**! You can see that we have declared the `UserCreatorRepository` as a
dependency, because the service can only interact with the database through the repository.

看一下**构造函数**！你可以看到我们声明了`UserCreatorRepository`作为一个依赖，因为只有service可以在repository和db间互动。

### Repositories 仓库

A repository is responsible for the data access logic, communication with database(s).

一个仓库服务于数据访问逻辑，也就是与数据库做交互。

There are two types of repositories: collection-oriented and persistence-oriented repositories. 
In this case, we are talking about **persistence-oriented repositories**, since these are better 
suited for processing large amounts of data.

有两种类型的仓库：面向集合和面向持久性仓库。我们讨论**面向持久性仓库**，因为对于处理较多数据时他们较优。

A repository is the source of all the data your application needs 
and mediates between the service and the database. A repository improves code maintainability, testing and readability by separating **business logic** 
from **data access logic** and provides centrally managed and consistent access rules for a data source. 
Each public repository method represents a query. The return values represent the result set 
of a query and can be primitive/object or list (array) of them. Database transactions must 
be handled on a higher level (service) and not within a repository.

仓库是你的app的数据资源，是在service和db中的交互。通过将业务逻辑和数据处理逻辑分开，仓库可以提高代码的可维护性，可测试性以及可阅读性并且提供了中心一致化管理规则。每个公共仓库方法都代表了一行query（译者注：SQL代码）。返回值则是query返回的数据，可以是一个对象，也可以是一个list，一个数组。数据库事务必须在更高级的层面处理（service）而不是在仓库内。

### Creating a repository 建立一个仓库

For this tutorial we need a test database with a `users` table.
Please execute this SQL statement in your test database.

创建`users`表，在`test`数据库中执行以下SQL代码。

```sql
CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(255) COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `email` varchar(255) COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `first_name` varchar(255) COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `last_name` varchar(255) COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `username` (`username`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

Add the database settings into: `config/settings.php`:

将数据库设置放在`config/settings.php`内。

```php
// Database settings
$settings['db'] = [
    'driver' => 'mysql',
    'host' => 'localhost',
    'username' => 'root',
    'database' => 'test',
    'password' => '',
    'charset' => 'utf8mb4',
    'collation' => 'utf8mb4_unicode_ci',
    'flags' => [
        // Turn off persistent connections
        PDO::ATTR_PERSISTENT => false,
        // Enable exceptions
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        // Emulate prepared statements
        PDO::ATTR_EMULATE_PREPARES => true,
        // Set default fetch mode to array
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        // Set character set
        PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci'
    ],
];
```

Insert a `PDO::class` container definition to `config/container.php`:

把`PDO::class`容器放在`config/container.php`内。

```php
PDO::class => function (ContainerInterface $container) {
    $config = $container->get(Configuration::class);

    $host = $config->getString('db.host');
    $dbname =  $config->getString('db.database');
    $username = $config->getString('db.username');
    $password = $config->getString('db.password');
    $charset = $config->getString('db.charset');
    $flags = $config->getArray('db.flags');
    $dsn = "mysql:host=$host;dbname=$dbname;charset=$charset";

    return new PDO($dsn, $username, $password, $flags);
},
```

From now on, PHP-DI will always inject the same PDO instance as soon as we declare PDO in a 
constructor as dependency.

从今以后，如果我们在构造函数中声明PDO为一个依赖，那么PHP-DI就会为我们注入相同的PDO作为依赖。

Create a new directory: `src/Domain/User/Repository`

新建文件夹：`src/Domain/User/Repository`

Create the file: `src/Domain/User/Repository/UserCreatorRepository.php` and insert this content:

新建文件`src/Domain/User/Repository/UserCreatorRepository.php`，拷贝以下代码：

```php
<?php

namespace App\Domain\User\Repository;

use App\Domain\User\Data\UserCreateData;
use PDO;

/**
 * Repository.
 */
class UserCreatorRepository
{
    /**
     * @var PDO The database connection
     */
    private $connection;

    /**
     * Constructor.
     *
     * @param PDO $connection The database connection
     */
    public function __construct(PDO $connection)
    {
        $this->connection = $connection;
    }

    /**
     * Insert user row.
     *
     * @param UserCreateData $user The user
     *
     * @return int The new ID
     */
    public function insertUser(UserCreateData $user): int
    {
        $row = [
            'username' => $user->username,
            'first_name' => $user->firstName,
            'last_name' => $user->lastName,
            'email' => $user->email,
        ];

        $sql = "INSERT INTO users SET 
                username=:username, 
                first_name=:first_name, 
                last_name=:last_name, 
                email=:email;";

        $this->connection->prepare($sql)->execute($row);

        return (int)$this->connection->lastInsertId();
    }
}

```

Note that we have declared `PDO` as a dependency, because the repository requires a database connection.

注意，我们声明了PDO作为一个依赖，因为这个仓库需要连接数据库。

The last part is to register a new route for `POST /users`.

最后一部分是为`POST /users`注册新路由。

Create a new action class in: `src/Action/UserCreateAction.php`:

新建`src/Action/UserCreateAction.php`：

```php
<?php

namespace App\Action;

use App\Domain\User\Data\UserCreateData;
use App\Domain\User\Service\UserCreator;
use Slim\Http\Response;
use Slim\Http\ServerRequest;

final class UserCreateAction
{
    private $userCreator;

    public function __construct(UserCreator $userCreator)
    {
        $this->userCreator = $userCreator;
    }

    public function __invoke(ServerRequest $request, Response $response): Response
    {
        // Collect input from the HTTP request
        $data = (array)$request->getParsedBody();

        // Mapping (should be done in a mapper class)
        $user = new UserCreateData();
        $user->username = $data['username'];
        $user->firstName = $data['first_name'];
        $user->lastName = $data['last_name'];
        $user->email = $data['email'];

        // Invoke the Domain with inputs and retain the result
        $userId = $this->userCreator->createUser($user);

        // Transform the result into the JSON representation
        $result = [
            'user_id' => $userId
        ];

        // Build the HTTP response
        return $response->withJson($result)->withStatus(201);
    }
}
```

In this example, we create a "barrier" between source data and output, 
so that schema changes do not affect the clients. For the sake of 
simplicity, this is done using the same method. In reality, you would 
separate the input data mapping and output JSON conversion into 
separate parts of your application.

在我们的例子中，我们在源数据和输出之间建立了一道屏障，这个变化不会影响到客户。为了简化，我们用了相同的方法。实际上，你或许会将数据映射和JSON输出放在app的不同地方。

Add the new route in `config/routes.php`:

在`config/routes.php`为我们的api新建路由

```php
$app->post('/users', \App\Action\UserCreateAction::class);
```

The complete project structure should look like this now:

现在完整的工程项目应该看起来像这样：

![image](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856544664_3344.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

Now you can test the `POST /users` route with [Postman](https://www.getpostman.com/) to see if it works.

测试下`POST /users`

If successful, the result should look like this:

如果一切正常，输入下图的4个Body参数后应该会返回user_id。

![image](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856545626_8210.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

## Deployment 部署

For deployment on a productive server, there are some important settings and security related things to consider.

在生产环境下部署需要注意一些有关安全的事项。

You can use composer to generate an optimized build of your application. 
All dev-dependencies are removed and the Composer autoloader is optimized for performance. 

你可以使用从composer来生成优化编译。

Run this command in the same directory as the project’s composer.json file:

在项目目录中运行如下代码作为`composer.json`文件

```
composer install --no-dev --optimize-autoloader
```

You don't have to run composer on your production server. Instead you should implement a [build pipeline](https://www.amazon.com/dp/B003YMNVC0) that creates
an so called "artifact". An artifact is an ZIP file you can upload and deploy on your production server. 
[selective-php/artifact](https://github.com/selective-php/artifact) is a tool to build artifacts from your source code.

你没有必要在运行环境下使用composer，相反地你应该实现一个流水线编译产物——artifact，产物。一个产物是一个ZIP压缩包，可以上传并部署到你的生产环境服务器，[selective-php/artifact](https://github.com/selective-php/artifact)是一个不错的工具。

For security reasons, you should switch off the output of all error details in production:

为了安全起见，你应该在生产环境下关闭所有错误输出：

```php
$settings['error_handler_middleware'] = [
    'display_error_details' => false,
];
```

If you have to run your Slim application in a sub-directory, you could try this library: [selective/basepath](https://github.com/selective-php/basepath)

如果你需要在服务器子目录下运行你的app，使用[selective/basepath](https://github.com/selective-php/basepath)库。

**Important**: It's very important to set the Apache `DocumentRoot` to the `public/` directory. 
Otherwise, it may happen that someone else could access internal files from the web. [More details](https://www.digitalocean.com/community/tutorials/how-to-move-an-apache-web-root-to-a-new-location-on-ubuntu-16-04)

将Apache的`DocumentRoot`设置为`public/`是很重要的，否则会产生一些权限的安全性问题：

`/etc/apache2/sites-enabled/000-default.conf`

```htacess
DocumentRoot /var/www/example.com/htdocs/public
```

**Tip:** Never store secret passwords in your git / SVN repository. 
Instead you could store them in a file like `env.php` and place this file one directory above your application directory. e.g.

不要把你的密码也上传到git / SVN仓库，你可以把他们放在一个文件里，例如下面这样，在app的目录的上一级。

```
/var/www/example.com/env.php
```

## Conclusion 结论

Remember the relationships: 

* Slim - To handle routing and dispatching 用来处理路由
* Single Action Controllers - To invoke the correct service method (domain) 用来调用domain的方法
* Domain - The core of your application 你的app的核心
* Services - To handle business logic 业务逻辑
* DTO - To carry data (no behavior) 数据处理
* Repositories - To execute database queries 数据库交互

## FAQ

### How to add JSON Web Token (JWT) / Bearer authentication?

Read this article: [Slim 4 - OAuth 2.0 and JSON Web Token Setup](https://odan.github.io/2019/12/02/slim4-oauth2-jwt.html) 

### CORS

Read more about [CORS](https://odan.github.io/2019/11/24/slim4-cors.html): 

### Where can I find the code on github?

The source code with more examples (e.g. reading a user) can be found here: <https://github.com/odan/slim4-tutorial>

A complete skeleton for slim 4 can be found here: <https://github.com/odan/slim4-skeleton>

### How to add a logger?

You could inject a logger factory, e.g. like the [LoggerFactory](https://github.com/odan/slim4-skeleton/blob/master/src/Factory/LoggerFactory.php)
The settings are defined [here](https://github.com/odan/slim4-skeleton/blob/master/config/defaults.php#L43). 

### I get a 404 (not found) error

Follow the instructions and define the correct base path with `$app->setBasePath('my-sub-directory/');`

If you have to run your Slim application in a sub-directory, you could try this library: [selective/basepath](https://github.com/selective-php/basepath)

### Error message: Callable (...) does not exist

Run `composer update` to fix it.

### How to add a database connection?

You can add a query builder as described here:

* [Slim 4 - Laminas Query Builder Setup](https://odan.github.io/2019/12/01/slim4-laminas-db-query-builder-setup.html)
* [Slim 4 - CakePHP Query Builder Setup](https://odan.github.io/2019/12/03/slim4-cakephp-query-builder.html)
* [Slim 4 - Eloquent Setup](https://odan.github.io/2019/12/03/slim4-eloquent.html)

### How to build assets with webpack?

* [Slim 4 - Compiling Assets with Webpack](https://odan.github.io/2019/09/21/slim4-compiling-assets-with-webpack.html)

## Read more

* [Slim 4 - Cheatsheet and FAQ](https://odan.github.io/2019/09/09/slim-4-cheatsheet-and-faq.html)

---

**版权所有**

本文作者为[Daniel Opitz](https://twitter.com/dopitz)，摘自https://odan.github.io/2019/11/05/slim4-tutorial.html，本文章仅为翻译。

Note: the content is the network collection written by [Daniel Opitz](https://twitter.com/dopitz) from https://odan.github.io/2019/11/05/slim4-tutorial.html and translated by me, if any infringement please contact me to delete!