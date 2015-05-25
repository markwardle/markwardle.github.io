
## Sessions

Sessions allow your application to store data pertaining to a specific user.  We will need to utilize sessions to keep our user logged in.  We don't need to add anything new packages since Symfony's HttpFoundation package also includes an object to handle sessions.

This section is going to be short since all we need to do is to subclass Symfony's session handler with a couple extra methods and to register it with the container.

Create a file named `Session.php` in the `src` directory with this code:

~~~~~~php
<?php

namespace YourNamespace;

use Symfony\Component\HttpFoundation\Session\Session as SymfonySession;
use YourNamespace\Model\User;

class Session extends SymfonySession
{
	public function setUser(User $user)
	{
		$this->set('user', [
			'name' => $user->name,
			'email' => $user->email,
		]);
	}

	public function userIsLoggedIn()
	{
		return null != $this->get('user');
	}
}
~~~~~~

Now add the session to the container in the app.php file (just after the code that registers the request object).

~~~~~~php
// Add the session to the container and alias it by class name
$container->singleton('session', 'YourNamespace\Session');
$container->add('YourNamespace\Session', function() use ($container) {
	return $container->get('session');
});
~~~~~~

Short and sweet.  Let's keep going.
