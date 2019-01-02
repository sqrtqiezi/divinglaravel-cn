# 写在前面

在了解 Laravel 如何自动发现包提供器和 Facades 之前，先浅谈一下 PHP 中的包的概念：

包是一段可以在多个项目上重用的代码。例如，[spatie/laravel-analytics](https://github.com/spatie/laravel-analytics) 是可以在任何一个 Laravel 项目中用来轻松地从 Google Analytics 中检索数据的一段代码。这个包托管在 GitHub 上，由 [Spatie](https://spatie.be/en) 团队的人去维护，他们会不断地对他们的包进行更新和错误修复。这个项目创建了 Composer，因此如果你在项目中使用了这个包，就算他们更新和修复了这个包，也不需要你从 Github 上复制新的代码。

> Composer 是 PHP 的一个依赖管理工具。它允许你声明项目所依赖的库并且为你管理（安装／更新）它们。-- getcomposer.org

Laravel 附带 `composer.json` 文件，可以在其中引入更多的包来扩展应用程序的功能。而你要做的就是在这个文件的 `require` 或者  `require-dev` 下面引入你需要的包，然后运行 `composer update`：

```json
{
    "require": {
        "spatie/laravel-analytics": "3.*",
    },
}
```

你也可以使用命令来达到同样的效果：

```bash
composer require spatie/laravel-analytics
```

运行这条命令之后，Composer 会拉取这个版本的包下载到你的 vendor 文件夹中。然后这个包里面所有的类和文件都会被加载到你的项目里面，你可以立即使用它。并且再次运行 `composer update` 时，Composer 会拉取这个包任何应用的更新，并自动更新到项目中的 vendor 文件夹。

一些 Laravel 包需要几个额外的步骤才能在 Laravel 的项目中使用：

* 注册服务提供者
* 注册 Aliases/Facades
* 发布资源

如果你查看 [Spatie 包的安装说明](https://github.com/spatie/laravel-analytics#installation)，你会发现在运行之前，你必须在项目配置中注册服务提供者和 Facade。[Taylor Otwell](https://twitter.com/taylorotwell) 认为这一步是不必要的，所以他与 [Dries Vints](https://twitter.com/driesvints) 合作，提出了一种能在任何一个你需要新的包或者你想要移除它们的时候自动注册服务提供者和 Facades 的方法。

关于这个可以查看 Taylor 在 [Medium](https://medium.com/@taylorotwell/package-auto-discovery-in-laravel-5-5-ea9e3ab20518) 上的 [公告](https://laravel-china.org/articles/4901/laravel-55-supports-packet-discovery-automatically)。

#### 什么是服务提供者和 facades ？

> 服务提供者负责将东西绑定到 Laravel 的服务容器中，并通知 Laravel 在哪里加载资源，比如视图、配置和本地文件。—— laravel.com 文档

你可以在 [官方文档]() 中阅读更多有关服务提供者的信息。

> Facades 为应用程序服务容器可用的类提供静态接口—— laravel.com 文档

你可以在 [官方文档](http://d.laravel-china.org/docs/5.4/facades) 中阅读更多有关 Facades 的信息。