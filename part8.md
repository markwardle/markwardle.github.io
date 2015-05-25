
## Templates and Controllers

PHP provides plenty of functionality on its own in terms of templating capability.  However, templating engines do provide facilities that make constructing HTML content easier.  If you want to use plain old php to generate your html, go right ahead and skip this section.

There are several templating options available for templating in PHP, but for this tutorial I have decided to use [Flow](https://github.com/nramenta/flow) which is based on the popular [Twig](http://twig.sensiolabs.org/) templating engine.  This tutorial won't cover all of Flow's features.  Instead it will demonstrate how to integrate a templating engine into your application.

By the end of this section, we should finally have some real content to display in the browser.

### Installing a templating engine

Naturally, the first thing we need to do is to include it in our application.

~~~~~~php
$ composer require flow/flow
~~~~~~

Now let's add some code to our `app.php` file. Let's put it after the code that registers the database, but before the code that registers the router.

~~~~~~php
// register the templating engine in the container
$container->singleton('templating_engine', function() {
	$flow = new Flow\Loader([
		'source' => dirname(__DIR__) . '/templates',
		'target' => dirname(__DIR__) . '/cache/templates'
	]);

	return $flow;
});

// alias the templating entine with its class name
$container->add('Flow\Loader', function() use ($container) {
	return $container->get('templating_engine');
});
~~~~~~

While we are in the `app.php` file we should add one more thing to it that we are going to need.  The container we are using allows us to register an interface that, for any class implementing this interface, will automatically call some action on it.  We are about to make a base controller class which will need this functionality for everything to work smoothly.  Add the following to the `app.php` file just after the code to create the container.

~~~~~~php
// Add the container to itself
$container->add('League\Container\ContainerInterface', $container);

// Add an inflector for the container aware interface
$container->inflector('League\Container\ContainerAwareInterface')
		  ->invokeMethod('setContainer', ['League\Container\ContainerInterface']);
~~~~~~

Now any object created with the container that implements the container aware interface will have the container set for it.

### Controllers

Before we play around with the templating engine, let's create a base controller class that our other controller classes can extend.  Create new directory named `Controller` within the `src` directory.  Within that directory create a file named `BaseController.php` with the following contents:

~~~~~~php
<?php

namespace YourNamespace\Controller;

use League\Container\ContainerAwareInterface;
use League\Container\ContainerAwareTrait;
use Symfony\Component\HttpFoundation\RedirectResponse;

abstract class BaseController implements ContainerAwareInterface
{
	// Let the library do the work for us
	use ContainerAwareTrait;

   /**
    * Renders a template with the data and returns it as a string
    */
	protected function render($template, $context = [])
	{
		$templatingEngine = $this->getContainer()
        						 ->get('templating_engine');
		$template = $templatingEngine->load($template);
		return $template->render($context);
	}

   /**
    * Returns a redirect response from a local path
    */
	protected function redirect($path)
	{
		$host = $this->getContainer()
					->get('request')->session
					->get('HTTP_HOST');

		$uri = $host . $uri;

		return new RedirectResponse($uri);
	}
}
~~~~~~

Excellent.  If we discover that there is functionality that all of our controllers need, we can always add it in this class.

Let's create our first controller.  We will make it to authentic users so they can log in to the site and write posts.  This class will wire together a lot of the work we have done so far.  Create an `AuthenticationController.php` file in the `Controller` directory with the following content:

~~~~~~php
<?php

namespace YourNamespace\Controller;

use Symfony\Component\HttpFoundation\Request;

use YourNamespace\Session;
use YourNamespace\Database;


class AuthenticationController extends BaseController
{
	public function login(Request $request, Session $session, array $data = [])
	{
		// make sure the user isn't already logged in
		// if they are, redirect to the home page
		if ($session->userIsLoggedIn()) {
			return $this->redirect('/');
		}

		// get previous form submission data if there is any
		$remembered = isset($data['formData']) ? $data['formData'] : [];
		$error = isset($data['formError']) ? $data['formError'] : '';
		$context = [
			'user' => $remembered,
			'error' => $error
		];

		return $this->render('authentication/login.flow.html', $context);
	}

	public function doLogin(Request $request, Session $session, Database $database)
	{
		// get the form inputs
		$email = $request->request->get('email');
		$password = $request->request->get('password');

		// look for a user with the given email
		$user = $database->findOne('user', ' email = ? ', [$email]);

		if (!$user) {
			return $this->login($request, $session, [
				'formData' => ['email' => $email],
				'formError' => 'There is no user with that email'
			]);
		}

		// check the password
		$isAuthenticated = $user->passwordIsCorrect($password);
		if (!$isAuthenticated) {
			return $this->login($request, $session, [
				'formData' => ['email' => $email],
				'formError' => 'That password is not valid'
			]);
		}

		// the user knew their password!
		$session->setUser($user);
		return $this->redirect('/');
	}

	public function logout(Session $session)
	{
		$session->invalidate();
		return $this->redirect('/login');
	}
}
~~~~~~

This controller has methods for loading a login form, handling an attempted login, as well as logging out the user.  The controller utilizes the User model class that we created as well as the Session class we created.  However, our app does not yet have any way to find these methods.  We need to add routes for these methods.

Open up your `config/routes.php` file, remove the existing contents, and replace them with the following:

~~~~~~php
<?php

$router->get('/login', 'YourNamespace\Controller\AuthenticationController::login');
$router->post('/login', 'YourNamespace\Controller\AuthenticationController::doLogin');
$router->get('/logout', 'YourNamespace\Controller\AuthenticationController::logout');
~~~~~~

This registers the three methods we made as routes.  If you go to these routes now they won't work, becauase we haven't created the template we need for the login page.  Let's do that now.

Before we create the login form template, let's create a base template.  Some templating platforms, including the one we are using in this tutorial, allow templates to extend one another allowing for a great amount of code reuse.

Create a file named `base.flow.html` in the `templates` directory in the root of the project.  Put this in it:

~~~~~~html
<!doctype html>
<html>
<head>
	<meta charset="utf-8">
	<title>{% block title %}{{ pageTitle or "My Blog" }}{% endblock %}</title>
	{% block styleSheets %}{% endblock %}
	{% block javaScripts %}{% endblock %}
</head>
<body>
	{% block body %}{% endblock %}
</body>
</html>
~~~~~~

That gets rid of some very basic boilerplate.  Now we can create our login form.  I'm not worrying too much about styles for this tutorial, so it ain't going to look good (sorry).

Create a new directory named `authentication` in the templates directory. In this directory create a file named `login.flow.html`.  In this file, we will extend the base template and add a form for the user to login with.

~~~~~~html
{% extends "../base.flow.html" %}

{% block title %}Login | MyBlog{% endblock %}

{% block body %}
	{% if error %}
		<div class="error">
			<p>{{error}}</p>
		</div>
	{% endif %}
	<form method="POST">
		<div class="input">
			<label for="email">Email:
				<input type="text" name="email" id="email" value="{{user.email}}">
			</label>
		</div>
		<div class="input">
			<label for="password">Password:
				<input type="password" name="password" id="password">
			</label>
		</div>
		<div class="input buttons">
			<input type="submit" value="Login">
		</div>
	</form>
{% endblock %}
~~~~~~

Awesome.  We can now see something real and even use it...kinda.  If you navigate to [localhost:8080/login](http://localhost:8080/login) (assuming your development server is running) you can see the login form.  You can even submit it, though you'll never be able to successfully login since there are no users.  Let's fix that.

Let's add two more methods to the controller we just made.

~~~~~~php
public function register(Session $session, $data = []) {
		// make sure the user isn't already logged in
		if ($session->userIsLoggedIn()) {
			return $this->redirect('/');
		}

		// get previous form submission data if there is any
		$remembered = isset($data['formData']) ? $data['formData'] : [];
		$error = isset($data['formError']) ? $data['formError'] : '';
		$context = [
			'user' => $remembered,
			'error' => $error
		];

		return $this->render('authentication/register.flow.html', $context);

	}

	public function doRegister(Request $request, Session $session, Database $database) {
		$email = $request->request->get('email');
		$name = $request->request->get('name');
		$password = $request->request->get('password');
		$passwordAgain = $request->request->get('password_again');

		$error = '';
		if (!$email) {
			$error = 'Email is required';
		} else if (!$name) {
			$error = 'Name is required';
		} else if (!$password) {
			$error ='Password is required';
		} else if ($password != $passwordAgain) {
			$error = 'Passwords do not match';
		} else if ($database->findOne('user', ' email = ? ', [$email])) {
			$error = 'A user with that email already exists';
		}

		if ($error) {
			return $this->register($session, [
				'formError' => $error,
				'formData' => [
					'name' => $name,
					'email' => $email
				]
			]);
		}

		$user = $database->dispense('user');
		$user->name = $name;
		$user->email = $email;
		$user->setPassword($password);
		$database->store($user);

		$session->setUser($user);

		return $this->redirect('/');
	}
~~~~~~

And now let's make the template for the registration form at `templates/authentication/register.flow.html`.

~~~~~~html
{% extends "../base.flow.html" %}

{% block title %}Register | MyBlog{% endblock %}

{% block body %}
	{% if error %}
		<div class="error">
			<p>{{error}}</p>
		</div>
	{% endif %}
	<form method="POST">
		<div class="input">
			<label for="email">Email:
				<input type="text" name="email" id="email" value="{{user.email}}">
			</label>
		</div>
		<div class="input">
			<label for="name">Your Name:
				<input type="text" name="name" id="name" value="{{user.name}}">
			</label>
		</div>
		<div class="input">
			<label for="password">Password:
				<input type="password" name="password" id="password">
			</label>
		</div>
		<div class="input">
			<label for="password">Password Again:
				<input type="password" name="password_again" id="password_again">
			</label>
		</div>
		<div class="input buttons">
			<input type="submit" value="Register">
		</div>
	</form>
{% endblock %}
~~~~~~

Finally, let's add the registration routes to the application as well as a stub home route.

~~~~~~php
$router->get('/register', 'YourNamespace\Controller\AuthenticationController::register');
$router->post('/register', 'YourNamespace\Controller\AuthenticationController::doRegister');

$router->get('/', function(){
	return 'made it home';
});
~~~~~~

You should now be able to register a user and login successfully.  Test it out.

At this point we have pretty much everything we need to finish our sample application.  The next section will show you how to get the app up on Heroku.  The main objective of this tutorial has been accomplished.  I may finish the app in another tutorial.

Make a commit before your continue.
