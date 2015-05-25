
## Hooking up the database

There are several tools that we could use to interact with the database.  For this tutorial we are going to use a tool called [RedBean](http://www.redbeanphp.com/) which provides a very simple way to create your database schema (it does it all for you!) and to interact with your database entities.

Additionally, we are going to use SQLite as our database.  This may not be the best choice for a production environment, but will work just fine for this tutorial.  If you want to use another database, just read RedBean's documentation to figure out how to do so.

### Installing a database component

Let's add RedBean to our project.

~~~~~~bash
$ composer require gabordemooij/redbean
~~~~~~

One challenge with this particular library is that it tends to assume that you are utilizing it through a static facade.  This is really convenient, but not ideal when we want to start testing our application.  It is possible to use RedBean without the facade, though also inconvenient.  To get around this problem, we are going to create our own database class and use PHP's magic `__call` method to automatically forward method calls to RedBean.

Create a `Database.php` file in the src directory with this code:

~~~~~~php
<?php

namespace YourNamespace;

class Database
{
   /**
	* Forwards calls to RedBean's facade
	*/
	public function __call($methodName, array $args)
	{
		$methodToCall = 'RedBeanPHP\R::' . $methodName;
		return call_user_func_array($methodToCall, $args);
	}
}
~~~~~~

Now, let's configure our database in our `app.php` file.  Put the following in in the file after thr code that creates the container but before the routing code:

~~~~~~php
// add the database to the container
$container->singleton('database', function() use ($container) {
	define( 'REDBEAN_MODEL_PREFIX', 'YourNamespace\\Model\\' )
	$db = new YourNamespace\Database();
	$dsn = 'sqlite:' . dirname(__DIR__) . '/db/myblog.sqlite3';
	$db->setup($dsn);
	return $db;
});

// alias the database with its class name
$container->add('YourNamespace\Database', function() use ($container){
	return $container->get('database');
});
~~~~~~

Great.  Let's make the models for our application.  Before that though, go ahead and make a command and then let's add another dependency to our application.  This one's a library that helps doing tasks with strings.  It's called [Stringy](https://github.com/danielstjules/Stringy).

~~~~~~bash
$ composer require danielstjules/stringy
~~~~~~

### Creating models

Create a directory named `Model` in the `src` directory.  We will first create a base model class that will extend RedBean's model class.  The reason we are doing this is so we can later add or modify functionality that applies to all of our models without having to modify RedBean's code.  Our model gives an example of this by setting timestamps for each model on create and update.

Put the following code in a `BaseModel.php` file in the Model directory:

~~~~~~php
<?php

namespace YourNamespace\Model;

use RedBeanPHP\SimpleModel;

class BaseModel extends SimpleModel
{
	protected function getDateTimeString($timestamp = null, $format = 'Y-m-d H:i:s')
	{
		return data($format, $timestamp ? $timestamp : time());
	}

   /**
    * This function is run every time a bean is saved
    */
	public function update()
	{
		// Set timestamps for every row of every table
		$timeString = $this->getDateTimeString();
		if (NULL === $this->bean->getID()) {
			$this->bean->created = $timeString;
		}
		$this->been->updated = $timeString;
	}
}
~~~~~~

Our simple application is only going to use three database tables, one for users, one for posts, and one for tags. Lets start with the users model.  Create a file called `User.php` in the `Model` directory with the following content:

~~~~~~php
<?php

namespace YourNamespace\Model;

class User extends BaseModel
{
	public function setPassword($password)
	{
		$this->bean->password = password_hash($password, PASSWORD_BCRYPT);
	}

	public function passwordIsCorrect($password)
	{
		return password_verify($password, $this->bean->password);
	}
}
~~~~~~

Next, we should to make a trait to generate slugs to avoid code repetition.  Create a new directory inside the `Model` directory named `Trait`.  In that directory, add a file named `Sluggable.php` with the following content:

~~~~~~php
<?php

namespace YourNamespace\Model\Trait;

use Stringy\Stringy as S;

trait Sluggable
{
	protected function verifySlug($slugifyField)
	{
		if (!$this->bean->slug) {
			$this->bean->slug = S::create($this->bean->$slugifyField)->slugify()->__toString();
		}
	}
}
~~~~~~

Now we can create our post and tag models in a `Post.php` and `Tag.php` file (respectively) within the `Model` directory.  The look very similar.

The post model:

~~~~~~php
<?php

namespace YourNamespace\Model;

class Post extends BaseModel
{
	// add the trait to the model
	use YourNamespace\Model\Trait\Sluggable;

	public function update()
	{
		$this->verifySlug('title');
		parent::update();
	}
}
~~~~~~

The tag model:

~~~~~~php
<?php

namespace YourNamespace\Model;

class Tag extends BaseModel
{
	// add the trait to the model
	use YourNamespace\Model\Trait\Sluggable;

	public function update()
	{
		$this->verifySlug('name');
		parent::update();
	}
}
~~~~~~

We've got our models all set up.  We will use them really soon, I promise.  There are just a few things we need to do first.  Commit your code and keep going.
