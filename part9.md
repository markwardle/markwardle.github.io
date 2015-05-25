
## Deploying to heroku

If you don't have a heroku account, get one.  Furthermore, heroku has excellent documentation on how to get things set up and deploy your application, so this will be a really quick version and assumes you can fill in the gaps.

To get started, run this command:

~~~~~~bash
$ composer require heroku/heroku-buildpack-php --dev
~~~~~~

Next, create a file named `Procfile` at the root of your application.  In this file put the following contents:

~~~~~~
web: vendor/bin/heroku-php-apache2 -C apache_app.conf public/
~~~~~~

Now we need the write heroku's equivalent of a .htaccess file.  Add a file named apache_app.conf at the root of the project with these contents:

~~~~~~php
RewriteEngine On

RewriteCond %{REQUEST_URI}::$1 ^(/.+)/(.*)::\2$
RewriteRule ^(.*) - [E=BASE:%1]

RewriteCond %{ENV:REDIRECT_STATUS} ^$
RewriteRule ^app\.php(/(.*)|$) %{ENV:BASE}/$2 [R=301,L]

RewriteCond %{REQUEST_FILENAME} -f
RewriteRule .? - [L]

RewriteRule .? %{ENV:BASE}/app.php [L]
~~~~~~

Next, let's initialize heroku.

~~~~~~bash
$ heroku create
~~~~~~

Next, we need to provision a heroku database.  No more sqlite.

~~~~~~bash
$ heroku addons:add heroku-postgresql:hobby-dev
~~~~~~

Modify the code that registers the database in the `app.php` file.

~~~~~~php
$container->singleton('database', function() use ($container) {
	define( 'REDBEAN_MODEL_PREFIX', 'Wardlem\\Model\\' );
	$db = new Wardlem\Database();
	$dbopts = parse_url(getenv('DATABASE_URL'));
	$dsn = 'pgsql:host=' . $dbopts['host'] . ';dbname=' . ltrim($dbopts["path"],'/') . ';port=' . $dbopts['port'];
	$db->setup($dsn, $dbopts['user'], $dbopts['pass']);
	return $db;
});
~~~~~~

Finally, commit all the changes, and push to Heroku.

~~~~~~bash
$ git add .
$ git commit -m "Ready for heroku"
$ git push heroku master
~~~~~~

And you are done.
