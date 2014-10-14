---
layout: post
title:  "Continuous delviery with Codeship, Laravel and Heroku"
date:   2014-10-14 12:20:51
---
With tools like Codeship available, setting up continuous delivery is now a possibility for businesses of any scale. Having just setup a straightforward continouos delivery arrangement for [www.careselector.com](http://www.careselector.com), I figured this would be a good time to document my efforts in using Codeship with a Laravel app.

Database configuration
------------------------
I use PostgreSQL, but I imagine the process for setting up MySQL or other databases is almost the same.

Luckily for us, the base Codeship VM on which it is going to run our tests will already have PostgreSQL installed, with some default databases allready created for us. Lets take advantage of this by using Laravel environments. Lets configure a 'codeship' environment by creating the directory 'app/config/codeship' and adding the following file: "app/config/codeship/database.php".

Because Codeship conveniently creates a default database called 'test', and stores the database user and password in enviroment variables, we can use this in our database config.

{% highlight php %}
// app/config/codeship/database.php
<?php

'connections' => array(

	'driver'   => 'pgsql',
        'host'     => 'localhost',
        'database' => 'test',
        'username' => $_ENV["PG_USER"],
        'password' => $_ENV["PG_PASSWORD"],
        'charset'  => 'utf8',
        'prefix'   => '',
        'schema'   => 'public',

);
{% endhighlight %}

Setting up the environment and environment variable
----------------------------------------------------
First we need to tell Codeship to set an environment variable 'APP_ENV' to 'codeship':
![Codeship enviroment example](/images/blog/codeship-environment-example.png "Codeship enviroment example")

Then we need to get laravel to run in the 'codeship' environment if the 'APP_ENV' variable is set. For this we need to open up "bootstrap/start.php" and set the environment based on the environment variable. This can be doen in many ways, but if you can use environment variables across the board, then this should suffice:

{% highlight php %}
// bootstrap/start.php
<?php

$env = $app->detectEnvironment(array(

	return $_ENV["APP_ENV"];

));
{% endhighlight %}

So now if we run our app in the Codeship VM, it will run in the 'codeship' environent.

Setup scripts
--------------
Before we are going to be able to run our tests on the Codeship VM, there are a few setup tasks that need to take place first. We are going to set up everything we need to run tests for Codeception, and Behat using the Selenium2 Webdriver.

Under the 'test' tab in codeship, lets enter our setup commands.

First lets specify our php version:
{% highlight bash %}
phpenv local 5.5
{% endhighlight %}

Install our Composer dependencies
{% highlight bash %}
composer install --prefer-source --no-interaction
{% endhighlight %}

Run Laravel migrations
{% highlight bash %}
php artisan migrate
{% endhighlight %}

Serve the application (and suppress any output from this, so that we can continue to use the terminal)
{% highlight bash %}
php artisan serve >/dev/null 2>&1 &
{% endhighlight %}

Start selenium server (selenium server must be in your repository,  it can be downloaded here: (http://docs.seleniumhq.org/projects/webdriver/)[http://docs.seleniumhq.org/projects/webdriver/])
{% highlight bash %}
java -jar selenium-server-standalone-2.42.2.jar >/dev/null 2>&1 &
{% endhighlight %}

Now that the server is set up, we can run our tests. Enter the following into the *test commands* box:
{% highlight bash %}
# Run codeception tests
vendor/bin/codecept run
# Run behat tests
vendor/bin/behat
{% endhighlight %}

We now have fully functioning continuous integration.

Deployment
------------
If the tests pass, we want codeship to deploy our app to Heroku. The first part of this is easy, in Codeship, under the deployment tab, simply select *Heroku* and enter your Heroku api key and app name. This doesnt quite cover everything though, as when we deploy we also want our Laravel migrations to run.

Fortunately for us, this is quite easy to set up. For this to work, we need to add another deployment. Under the deployment tab, click on *script* and enter the following:

{% highlight bash %}
heroku_run php artisan migrate
{% endhighlight %}

And there we have it. A fully funcitoning Continuos Deployment setup using Laravel, Codeship and Heroku.

