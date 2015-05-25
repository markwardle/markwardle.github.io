
## Routing

Because we are using a single point of entry into our application, we will need to route incoming url paths to specific actions within our application.  This is called routing.  There are plenty of routers out there, but for this application we will utilize one maintained by the [The League of Extraordinary Packages](http://route.thephpleague.com/).

### Installing and configuring the router

To install the router using Composer, enter the following command in your terminal from the your application root.

~~~~~~bash
$ composer require league/route
~~~~~~

Composer will then fetch the library and all of its dependencies.  Since we already have our autoloader set up, there is nothing further we need to do to start using the package.

If you read the documentation for the component, you will notice that it allows for several strategies for dispatching routes.  For this tutorial we are going to write our own strategy which is a combination of the provided method argument strategy and uri strategy.

The route component requires that the dispatching strategy implements `Leauge\Route\Strategy\StrategyInterface`.  It also provides an abstract class that we can extend which will make writing our strategy really easy.

Create a new folder in the projects `src` directory called `Routing`.  Within that directory create a file called `Strategy.php`.  In this file we will create our routing strategy class.  You should choose a base vendor namespace that you will use for your project.  In the code when you see 'YourNamespace' in a namespace, you should replace it with whatever you want.  I use 'Wardlem'.

Add the following code to the file.

~~~~~~php
<?php

namespace YourNamespace\Routing;

use League\Route\Strategy\AbstractStrategy;
use League\Route\Strategy\StrategyInterface;

class Strategy extends AbstractStrategy implements StrategyInterface
{

	protected function processVars(array $vars) {

		$container = $this->getContainer();
		$router = $container->get('router');

		foreach($vars as $key => $value) {
			if ($router->hasProcessor($key)) {
				$processor = $router->getProcessor($key);
				$processedValue = $container->call($processor, [$key => $value]);
				$vars[$key] = $processedValue;
			}
		}

		return $vars;
	}

	public function dispatch($controller, array $vars)
	{

		$vars = $this->processVars($vars);

        if (is_array($controller) && is_string($controller[0])) {
			$controller[0] = $this->getContainer()->get($controller[0]);
		}

		$response = $this->getContainer()->call($controller, $vars);
		return $this->determineResponse($response);
	}
}
~~~~~~

What this strategy does is that it allows use to create handler functions (i.e. controllers) for our routes that will have their methods automatically resolved according to class type hints and parameter names.  It will also preprocess route parameters for us so that we don't have to do it manually in our controllers. This will make more sense in a moment.

Additionally, we are going to extend the router so that we can add some custom functionality.  When you need custom functionality, you should never alter the library code.  You should always create your own class and extend it.

Create a `Router.php` file in the same directory as the strategy with the following contents.

~~~~~~php
<?php

namespace YourNamespace\Routing;

use League\Route\RouteCollection;

class Router extends RouteCollection
{
	protected $processors = [];

	public function addProcessor($routeKey, callable $processor)
	{
		$this->processors[$routeKey] = $processor;
	}

	public function hasProcessor($routeKey)
	{
		return isset($this->processors[$routeKey]);
	}

	public function getProcessor($routeKey)
	{
		return $this->processors[$routeKey];
	}
}
~~~~~~

We need to create the routing code in our `app.php` file.  At the end of the file, add the following:

~~~~~~php
// create the router and add it to the container
$router = new YourNamespace\Routing\Router($container);
$router->setStrategy(new YourNamespace\Routing\Strategy());
$container->add('router', $router);
$container->add('YourNamespace\Routing\Router', $router);

// load the routes for the application
require dirname(__DIR__) . '/config/routes.php';

// dispatch the request to the router and return the response to the client
$dispatcher = $router->getDispatcher();
$request = $container->get('request');
$response = $dispatcher->dispatch($request->getMethod(), $request->getPathInfo());
$response->send();
~~~~~~

What this code does is create a router, adds the router to the container (just in case some other code in our application needs it for whatever reason), requires a file that will contain our routes, dispatches the request based on the http method and URI path, and finally sends the result to the client.  We haven't made our routing file yet, so we should do it.  We could put all of our routes in the `app.php` file, but to keep things a bit cleaner, we will do it in a separate file.

Create a file named `routes.php` in the `config` directory.  Add the following test code:

~~~~~~php
<?php

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

$router->addProcessor('name', function($name) {
	return strtoupper($name);
});

$router->addRoute('GET', '/', function() {
	return 'This is the home page!';
});

$router->addRoute('GET', '/hello/{name}', function(Request $request, $name) {
	$response = 'Hello there ' . $name . '.<br>';
	$response .= 'The current path is ' . $request->getPathInfo();
	return new Response($response);
});
~~~~~~

That should be all we need for our demonstration.  However, PHP does not yet have a way to find our strategy class.  Let's fix that.

### Autoloading our files

Composer makes it really easy for us to add our class files to its autoloader.  Open up the `composer.json` which is at the root of our project.  We should already have some content in there which composer has generated.  We will add our autoloading configuration after the require object under an "autoload" key.  Edit the file so that it looks like this:

~~~~~~json
{
    "require": {
        "league/container": "^1.3",
        "symfony/http-foundation": "^2.6",
        "league/route": "^1.1"
    },
    "autoload": {
    	"psr-4": {
    		"YourNamespace\\": "src/"
    	}
    }
}
~~~~~~

Our files will be autoloading using the strategy defined in [psr-4](http://www.php-fig.org/psr/psr-4/) outlined by the [PHP Framework Interop Group](http://www.php-fig.org) (FIG).  Fortunately, Composer knows exactly what it needs to do using this strategy to load our files.

We need Composer to regenerate its autoload script so that it includes our classes.

~~~~~~php
$ composer dump-autoload
~~~~~~

Our classes will now be automatically loaded for us.

### Testing things out

If your development server isn't running, go ahead and run it.  Navigate to [localhost:8080](http://localhost:8080).  You should see 'This is the home page' in the browser. Next try going to [localhost:8080/hello/somebody](http://localhost:8080/hello/somebody).  You should see something like this:

![Hello Somebody](images/hello-somebody2.png)

We have a router. Awesome.  Don't forget to commit your changes!
