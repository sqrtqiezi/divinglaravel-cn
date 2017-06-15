# Before The Dive

Before getting into how Laravel auto-discovers package providers and facades, let's first have a shallow dive on the concept of packages in PHP:

A package is a piece of code that you can reuse in multiple projects, [spatie/laravel-analytics](https://github.com/spatie/laravel-analytics) is a piece of code that you can use in any of your laravel projects to have an easy way to retrieve data from Google Analytics, such package is hosted on GitHub and is well maintained by the fine folks at [Spatie](https://spatie.be/en) and they constantly release new updates and bug fixes on their packages, if you use this package in your project you'd want to have these updates and fixes once they're released and not have to worry about copying the new code from Github, for that [Composer](https://getcomposer.org/) was created.

> Composer is a tool for dependency management in PHP. It allows you to declare the libraries your project depends on and it will manage (install/update) them for you. -- getcomposer.org

Laravel is shipped with a `composer.json` file where you can require more packages to extend the functionality of your application, all you have to do is include the package you want under the `require` or `require-dev` section of that file and then run `composer update`:

```json
{
    "require": {
        "spatie/laravel-analytics": "3.*",
    },
}
```

You can also use the following command that'll have the same effect:

```bash
composer require spatie/laravel-analytics
```

At this point Composer did its job and pulled the version of that package that you want and downloaded it to your `vendor` directory, now all the classes and files of this package is loaded into your project and you can use it right away, and every once in a while you can run `composer update` again and Composer will fetch any updates applied to this package and automatically update the files in your projects `vendor` directory.

Some Laravel packages require a few extra steps for it to be usable in a Laravel project:

* Register service providers
* Register Aliases/Facades
* Publish assets

If you take a look at the [installation instructions of Spatie's package](https://github.com/spatie/laravel-analytics#installation) you'll find that you have to register a service provider and a Facade in your project configuration before you're good to go, this step was identified by [Taylor Otwell](https://twitter.com/taylorotwell) as un-necessary so he teamed up with [Dries Vints](https://twitter.com/driesvints) and came up with a way to auto-register Service Providers and Facades whenever you require a new package and also remove them if you decide to remove the package at any time.

Check out Taylor's announcement of the feature [on Medium](https://medium.com/@taylorotwell/package-auto-discovery-in-laravel-5-5-ea9e3ab20518).

### What are service providers and facades?

> A service provider is responsible for binding things into Laravel's service container and informing Laravel where to load package resources such as views, configuration, and localization files. -- laravel.com Docs

You can read more about the Service Providers on the [official documentation](https://laravel.com/docs/5.4/providers).

> Facades provide a "static" interface to classes that are available in the application's service container -- laravel.com Docs

You can read more about the Service Providers on the [official documentation](https://laravel.com/docs/5.4/facades).