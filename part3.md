# How to properly set up a PHP application with composer and deploy it to Heroku - Part 3


## Inversion of Control

At the heart of almost every modern PHP framework lies an Inversion of Control (a.k.a Dependency Injection) Container.  The merits of utilizing such a container are many and can be discovered using a simple Google search.  The most obvious benefits that you will see are a reduction in boilerplate code when it comes to resolving your objects dependencies as well as increasing the testability of your code.

### Installing a container

The first dependency that we will require with composer will be our dependency injection container.  That are plenty of good options when it comes to dependency injection containers for PHP.  For this tutorial, I have decided to use a [container](http://container.thephpleague.com/) created by *The League of Extraordinary Packages*.

To install the container we will run the following command in the terminal at the root of our project:

~~~~~~bash
$ composer require league/container
~~~~~~

This command will do several things.  Firstly, it will create a `composer.json` file at the root which will keep track of the dependencies we have installed in our application.  If you know what you are doing, feel free to toy around with this file.  Secondly, will also create a `composer.lock` file which is used internally by composer.  You will never need to mess with this file, just leave it where it is.  Thirdly it will create a `vendor` directory where it will store the files for everything that we have installed.  Feel free to take a look inside if you wish, though these files should not be toyed around with.  In the vendor directory you will find our newly installed container component.

### Creating a .gitignore file

Before you commit anything, we should create a `.gitignore` file in the project root directory.  This will prevent git from tracking files that we don't want it to upload.  We don't need to commit any of the files that Composer installs because anyone with Composer and our composer.json file can reinstall them as needed.  Put the following in your ignore file:

~~~~~~
vendor
cache
~~~~~~

We should run a commit real quick.

~~~~~~shell
$ git add .gitignore
$ git add composer.json
$ git add public/app.php
$ git commit -m "Added league/container as an application dependency."
~~~~~~

### Autoloading

In case you are unaware, PHP has a pretty handy tool called 'autoloading' which will automatically find files based on their name and load them for you rather than having to write our own `require` statements.  Usually, this is done with one or more autoloading functions which are registered when the application starts up.  However, Composer handles autoloading for all of the dependencies we install, and can even handle autoloading of our own files for us.  All we have to do is to require a single file which Composer generates every time the projects dependencies change.  We will require it at the top of our app.php file.  Open it up and add the following code:

~~~~~~php
<?php

// load the autoloader
require_once dirname(__DIR__) . "/vendor/autoload.php";
~~~~~~

With that done, all of our dependency classes will be loaded automatically when we need them.  We will want our own files autoloaded as well, but we will deal with that later.
