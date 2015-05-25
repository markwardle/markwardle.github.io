# How to properly set up a PHP application with composer and deploy it to Heroku - Part 4


## HTTP Request and Response

PHP provides information about the current request in the `$_POST`, `$_GET`, and other global variables.  While this is convenient, we want to deal with all this in a more object oriented fashion.  A popular component for doing this is the [HttpFoundation](http://symfony.com/doc/current/components/http_foundation/index.html) component created by the team behind the Symfony framework.  Let's install it.

~~~~~~bash
$ composer require symfony/http-foundation
~~~~~~

### Creating a dependency definition

The request object should only be created once by our application.  To facilitate this we will use our dependency injection container. Add the following code to you your app.php file after the require statement for the autoloader.

~~~~~~php
use League\Container\Container;
use Symfony\Component\HttpFoundation\Request;

// create the IoC container
$container = new Container;

// register the singleton request object in the container
$container->singleton('request', Request::createFromGlobals());

// add an alias for the request in the container
$container->add('Symfony\Component\HttpFoundation\Request', function() use ($container) {
	return $container->get('request');
});
~~~~~~

### Setting up a test server

Now that we have a little bit going on, it is probably a good time to see some action.  If you have apache installed you are more than welcome to set it up to serve your application.  We will be doing this later and adding a `.htaccess` file to control the entry to our application. Otherwise, you will probably want to set up a development server.  We will do this using PHP's built in server (This requires PHP 5.4+).

We will write the server's login in a server.php file in the root of our project.  When we set up the live server, we will want apache to serve an files in the public directory if accessed directly, otherwise we will load the app.php file and let it do its work.  Ad the following code (borrowed from the Laravel project) to a newly created server.php file in your project's root directory.

~~~~~~php
<?php
$uri = urldecode (
	parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH)
);

if ($uri !== '/' && file_exists(__DIR__ . '/public' . $uri)) {
	return false;
}
require_once __DIR__ . '/public/app.php';
~~~~~~

Great.  That's all our server needs to do a good enough job of mimicing Apache's behavior for our purposes.

Now, we need to put something interesting on the browser.  At the bottom of your public/app.php file, add the following snippet:

~~~~~~php
$request = $container->get('request');
printf("Received request with a path of '%s'.", $request->getPathInfo());
~~~~~~

We can now start the server.  In your project root run the following command and then navigate to localhost:8080/any/path/you/like:

~~~~~~php
$ php -S localhost:8080 -t public/ server.php
~~~~~~

You should see something like this:

![Server Success](images/server-success.png)

When you are satisfied, go ahead and delete the last two lines we added to the `app.php` file.  We won't be needing them.

You can stop the server any time by entering `Ctrl-C` in the terminal.  You don't need to though, since all the files will be reloaded on each request and all your changes will be visible.

Let's make a commit.

~~~~~~bash
$ git add .
$ git commit -m "Added an HTTP library and a development server script."
~~~~~~
