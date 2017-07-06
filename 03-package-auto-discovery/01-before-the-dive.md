# 写在前面

Before getting into how Laravel auto-discovers package providers and facades, let's first have a shallow dive on the concept of packages in PHP:

在了解 Laravel 如何自动发现包提供器和 Facades 之前，首先要对 PHP 中的包的概念进行浅析：

A package is a piece of code that you can reuse in multiple projects, [spatie/laravel-analytics](https://github.com/spatie/laravel-analytics) is a piece of code that you can use in any of your laravel projects to have an easy way to retrieve data from Google Analytics, such package is hosted on GitHub and is well maintained by the fine folks at [Spatie](https://spatie.be/en) and they constantly release new updates and bug fixes on their packages, if you use this package in your project you'd want to have these updates and fixes once they're released and not have to worry about copying the new code from Github, for that [Composer](https://getcomposer.org/) was created.

包是一段可以在多个项目上重用的代码。[spatie/laravel-analytics](https://github.com/spatie/laravel-analytics) 是可以在任何一个 Laravel 项目中用来轻松地从 Google Analytics 中检索数据的一段代码。这样的包托管在 GitHub 上，由 [Spatie](https://spatie.be/en) 团队的人去维护，他们不断地对他们的包进行更新和错误修复。由于创建了这个项目的 Composer，因此如果你在项目中使用了这个包，在他们更新和修复这个包时，你不需要从 Github 上复制新的代码。

> Composer is a tool for dependency management in PHP. It allows you to declare the libraries your project depends on and it will manage (install/update) them for you. -- getcomposer.org
>
> Composer 是 PHP 的一个依赖管理工具。它允许你声明项目所依赖的库并且为你管理（安装／更新）它们。

Laravel is shipped with a `composer.json` file where you can require more packages to extend the functionality of your application, all you have to do is include the package you want under the `require` or `require-dev` section of that file and then run `composer update`:

Laravel 配有一个 `composer.json` 文件，可以在其中引入更多的包来扩展应用程序的功能。而你要做的就是在这个文件的 `require` 或者  `require-dev` 下面引入你需要的包，然后运行 `composer update`：

```json
{
    "require": {
        "spatie/laravel-analytics": "3.*",
    },
}
```

You can also use the following command that'll have the same effect:

你也可以使用命令来达到同样的效果：

```bash
composer require spatie/laravel-analytics
```

At this point Composer did its job and pulled the version of that package that you want and downloaded it to your `vendor` directory, now all the classes and files of this package is loaded into your project and you can use it right away, and every once in a while you can run `composer update` again and Composer will fetch any updates applied to this package and automatically update the files in your projects `vendor` directory.

运行这条命令之后，Composer 会拉取这个版本的包下载到你的 vendor 文件夹中。然后这个包里面所有的类和文件都会被加载到你的项目里面，你可以立即使用它。并且每一次再次运行 `composer update`  时，Composer 会拉取这个包任何应用的更新，并自动更新到项目中的 vendor 文件夹。



Some Laravel packages require a few extra steps for it to be usable in a Laravel project:

一些 Laravel 包需要几个额外的步骤才能在 Laravel 的项目中使用：

* Register service providers
* Register Aliases/Facades
* Publish assets
* 注册服务提供者
* 注册 Aliases/Facades
* 发布资源

If you take a look at the [installation instructions of Spatie's package](https://github.com/spatie/laravel-analytics#installation) you'll find that you have to register a service provider and a Facade in your project configuration before you're good to go, this step was identified by [Taylor Otwell](https://twitter.com/taylorotwell) as un-necessary so he teamed up with [Dries Vints](https://twitter.com/driesvints) and came up with a way to auto-register Service Providers and Facades whenever you require a new package and also remove them if you decide to remove the package at any time.

如果你查看 [Spatie 包的安装说明](https://github.com/spatie/laravel-analytics#installation)，你会发现在运行之前，你必须在项目配置中注册服务提供者和 Facade。[Taylor Otwell](https://twitter.com/taylorotwell) 认为这一步是不必要的，所以他与 [Dries Vints](https://twitter.com/driesvints) 合作，提出了一种能在任何一个你需要新的包或者你想要移除它们的时候自动注册服务提供者和 Facades 的方法。



Check out Taylor's announcement of the feature [on Medium](https://medium.com/@taylorotwell/package-auto-discovery-in-laravel-5-5-ea9e3ab20518).

可以查看 Taylor 在 [Medium](https://medium.com/@taylorotwell/package-auto-discovery-in-laravel-5-5-ea9e3ab20518) 上的 [公告](https://laravel-china.org/articles/4901/laravel-55-supports-packet-discovery-automatically)。

#### What are service providers and facades?

什么是服务提供者和 facades ？

> A service provider is responsible for binding things into Laravel's service container and informing Laravel where to load package resources such as views, configuration, and localization files. -- laravel.com Docs
>
> 服务提供者负责将东西绑定到 Laravel 的服务容器中，并通知 Laravel 在哪里加载资源，比如视图、配置和本地文件。—— laravel.com 文档

You can read more about the Service Providers on the [official documentation](https://laravel.com/docs/5.4/providers).

你可以在 [官方文档]() 中阅读更多有关服务提供者的信息。

> Facades provide a "static" interface to classes that are available in the application's service container -- laravel.com Docs
>
> Facades 为应用程序服务容器可用的类提供静态接口—— laravel.com 文档

You can read more about the Service Providers on the [official documentation](https://laravel.com/docs/5.4/facades).

你可以在 [官方文档](http://d.laravel-china.org/docs/5.4/facades) 中阅读更多有关 Facades 的信息。