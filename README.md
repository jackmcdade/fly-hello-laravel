# Fly Hello Laravel

A minimal Laravel application for Fly.io.

## Run it locally

You will need PHP 8+. You can check the version using `php --version`. And [composer](https://getcomposer.org/).

1. Clone this repo
2. Duplicate `.env.example` naming it `.env`
3. Run `composer install` to install its dependencies
4. Run `php artisan key:generate` to generate a new secret key
5. Run `php artisan serve` to run a local development server

You should be able to visit `http://localhost:8000` and see the home page.

## Deploy it to Fly

1. Edit the provided `fly.toml` so it has your choice of app name and URL instead of these:

```toml
app = "fly-hello-laravel"

APP_URL = "https://fly-hello-laravel.fly.dev"
```

2. Run `fly launch`. When it gets to the point asking if you want to deploy now, say **No**. Why? Because in production you need a secret **APP_KEY**. Without it the app will return an error like:

> No application encryption key has been specified. "exception":"[object] (Illuminate\\Encryption\\MissingAppKeyException"

3. Set that _APP_KEY_ by running `fly secrets set APP_KEY=the-value-from-your-env-file`
4. Run `fly deploy`

You should be able to visit `https://your-app-name.fly.dev` and see the Laravel demo home page.

## Build, deploy and run any Laravel application on Fly

In this guide we'll learn how we packaged this Laravel application into an image ready to deploy to Fly's global application platform.

This is _slightly_ more complicated than it is for other runtimes since PHP does not include a web server. We need to add one. Here we use nginx. And so we need to keep both it _and_ PHP running. We do that using [supervisor](https://github.com/Supervisor/supervisor).

### Create a new Laravel application

If you _already_ have a Laravel application you would like to deploy, skip this step.

There are different approaches to creating a brand new Laravel application depending on your OS. Here we've used the [laravel-installer](https://laravel.com/docs/9.x#the-laravel-installer) approach. It assumes you have [composer](https://getcomposer.org/doc/00-intro.md#system-requirements) and PHP already installed:

```
composer global require laravel/installer

laravel new example-app

cd example-app

php artisan serve
```

You should be able to visit `http://localhost:8000` and see the default Laravel home page.

### Modify your application

Now that you have _a_ Laravel application working locally, you need to add some files in order to run it on Fly. This method uses [supervisor](https://github.com/Supervisor/supervisor) to keep nginx and PHP running. There are other ways to do this though. See: [https://fly.io/docs/app-guides/multiple-processes/](https://fly.io/docs/app-guides/multiple-processes/).

To make _this_ approach work you need to add four things:

1. a `Dockerfile`
2. a `.dockerignore`
3. a `/docker` folder which contains configuration files for PHP, nginx, and supervisor
4. a `fly.toml` that tells Fly what type of application you have (its port, protocol, and so on). Fly can generate this for you, however we can provide our own containing the environment variables we know that we'll need

You can just copy the files we've provided (adjusting each file depending upon your application's requirements). And then skip ahead to **Add a fly.toml** below.

But if you'd like to know why we made those changes, please continue:

#### Add a Dockerfile

The `Dockerfile` tells Fly the dependencies that need to be installed in order for the application to run. If you take a look at the one we added, you can see each step is commented. You can adjust it as desired. For example you may not be using Laravel's _mix_ for static assets. In which case you could remove the references to _node_ and _npm_.

This example uses a basic base image in order to install only what we need and make clear what is going on at each step. To keep the size small it uses Alpine Linux rather than a much larger base OS, such as Ubuntu.

As mentioned above, some runtimes contain a web server. PHP does not. So we need nginx to proxy requests to it. But now we have multiple processes to keep running (`php-fpm` and `nginx`). To do that we use `supervisor`. So we install that too.

#### Add a .dockerignore

If we simply added a `Dockerfile` with no `.dockerignore` file, it would include _all_ the files in our application's folder. That would include ones that we certainly don't want being deployed (like `.env`) and ones we don't need deployed (like the `node_modules` folder). It makes clear the files and folders we _do_ want included.

It is a personal preference to use a _deny-unless-admitted_ approach for safety. If we add a file or folder in future, we can choose whether we want it deployed by adding it to the `.dockerignore`.

#### Add a docker folder

This contains the various configuration files for PHP, nginx and supervisor to tell them what to do.

During the build they get moved to where the various processes expect them to be. For example in the `Dockerfile` you will see we move the custom nginx condfguration file to where nginx expects it to be by using this command `RUN mv docker/nginx.conf /etc/nginx/nginx.conf`.

These configuration files will likely vary depending on the functionality your application needs and the size of Fly vm you choose. For example you could adjust the request timeouts or the amount of memory allocated to PHP.

What are the files in this folder?

The `docker/supervisor.conf` file governs what processes `supervisor` starts and/or keeps running. In this case we have a group of programs. At a minimum we need `nginx` and `php-fpm` to be running. You will see they are set to _autorestart_. That's important as we need them both to be running for the application to serve requests.

Their logs could be sent to a local file. However we don't really want to have to manually SSH in to every vm to see the log files. Far better they output to `/dev/stdout` or `/dev/stderr` as then we can access them by simply using the `fly logs` command.

Since our application does not use queues, notifications or scheduling, those sections are commented out in our `supervisor.conf` file. They are just there to give an indication of how they _may_ work.

The `docker/nginx.conf` provides a complete nginx configuration. This is based on the Alpine Linux [nginx docs](https://wiki.alpinelinux.org/wiki/Nginx). You might want a simpler one, such as [Laravel's example nginx conf](https://laravel.com/docs/9.x/deployment#nginx). Note again how we change its default behaviour of logging to a local file to log to stdout/stderr:

```
#access_log  /var/log/nginx/access.log main;
#error_log /var/log/nginx/error.log;
access_log /dev/stdout main;
error_log /dev/stdout;
```

The `docker/php.ini`, `docker/php-fpm.conf` and `docker/app.conf` determine how PHP runs.

In `docker/php.ini` we only included the options we actually want to override. For example we applied some common options recommended for security, such as `expose_php = Off`.

The `docker/php-fpm.conf` and `docker/app.conf` are complete files taken from the original installation of PHP-FPM. Using a complete file lets us to see exactly what default values it uses. That's important because there are some that need to be changed to run a PHP application in a container.

Of those two files, the `docker/php-fpm.conf` does not need to be changed much. Again, we simply change where its errors are output:

```
;error_log = log/php8/error.log
error_log = /dev/stderr
```

The `docker/app.conf` includes the most changes. The default PHP-FPM installation calls this file `www.conf` using a `[www]` pool. We changed it to `[app]`. So this replaces the `www.conf` file you would otherwise see.

If you look through the file you can see the commented-out "OLD" values and their replacements. We've changed the user. We've changed the port to a socket. We also change the value of two important variables which are not obvious: `catch_workers_output` and `clear_env`:

```
catch_workers_output = yes
clear_env = no
```

##### catch_workers_output

What does _this_ do? Again, it's for logging. If have used Laravel's logger helper `logger()->debug('Test');` or facade `Log::debug('Test');` when you run your app locally, you may log to a file (`laravel.log`) or you may output to your terminal. Those log lines can be extrememly useful for debugging. If you left the default value of `catch_workers_output = no` in this conf file, that output would not be caught. So you would never see it. Catching the output allows it to be piped to the master process, and on to Fly.

##### clear_env

What does _this_ do? This lets PHP-FPM access the environment variables provided by the system. If you leave this as its default _yes_ you will find Laravel is not happy. It can't access all the environment variables you've either set within the `fly.toml` file (we'll come on to that in a moment) or by using `fly secrets`. And those environment variables are _very_ important, controlling things such as the application environment and secret key.

#### Add a fly.toml

If you have deployed any application to Fly before, you will recognise this. The `fly.toml` file tells Fly about your application, such as what ports it should use, the protocol and health-checks. One is generated for you by the Fly CLI however you may prefer to have one already if you know the settings you need.

Our `fly.toml` includes some standard settings, such as exposing port `443` and `80` to the outside world, and have our app listening on port `8080`. It has a http healthcheck on `/` which (if all is well) will return a `200` status code. Notice also the `[env]` section which contains environment variables. This contains any variable whose value is not secret.

If using our `fly.toml` you will need to update the application name from _fly-hello-laravel_ in two places:

```toml
app = "fly-hello-laravel"

APP_URL = "https://fly-hello-laravel.fly.dev"
```

**Note:** You should set `APP_DEBUG` as false when running Laravel in production. But we _don't_ set that here within the `fly.toml`. Why? Currently it seems that that value is parsed into a string. This can be confirmed using `gettype(env('APP_DEBUG'))` within Laravel. It returns _string_. And the string _"false"_ is _truthy_. And so debgging remains on. In Laravel's `config/app.php` the default value of _APP_DEBUG_ should be false and so it is left as false _unless_ we set it as true.

We also haven't made use of the `[[statics]]` option to [offload serving static assets](https://fly.io/docs/reference/configuration/#the-statics-sections) to Fly. That is because we are using nginx which is already a very efficient way to serve static files. However you could add that if you prefer.

Due to the way Laravel caches files, we don't use the option to cache configuration. We want to still have access to environment variables set at runtime.

You should also make sure you do not deploy with cached routes. Why? If a different _APP_KEY_ was used (as it will likely be) you will get a Laravel error complaining that:

> Your serialized closure might have been modified or it's unsafe to be unserialized

... and your application won't run. We delete the cache in the `Dockerfile` as part of the build.

### Optional: static assets

You do not have to use Laravel's [mix](https://laravel.com/docs/9.x/mix) to compile and minify static assets however if you want to, you will need to install it and run it:

```
npm install

npm run dev
```

### Deploy your application to Fly

If you haven't already done so, [install the Fly CLI](https://fly.io/docs/getting-started/installing-flyctl/) and then [log in to Fly](https://fly.io/docs/getting-started/log-in-to-fly/).

To launch the app, run `fly launch` from the application's directory.

The CLI will spot the existing `fly.toml`:

```
An existing fly.toml file was found for app fly-hello-laravel
? Would you like to copy its configuration to the new app? (y/N)
```

Type _y_ (yes).

The CLI will spot the `Dockerfile`:

```
Scanning source code
Detected a Dockerfile app
```

You'll be asked to give the app a name.

You'll be prompted to choose an organization. They are used to share resources between Fly users. Since every Fly user has a personal organization, let's pick that.

You'll be asked for the region to deploy the application in. Pick one closest to you for the best performance. That should already be selected.

It will ask if you want a database. In this case type _n_ (no).

It will then ask if you want to deploy now. Say **No**. Why? In production your application needs to have a secret key set. If you were to deploy _now_ you would see errors in the logs along the lines of:

> No application encryption key has been specified. "exception":"[object] (Illuminate\\Encryption\\MissingAppKeyException"

You can get that secret value for `APP_KEY` from your `.env` file (or you can generate a new one using `php artisan key:generate`).

Run `fly secrets set APP-KEY=the-value-of-the-secret-key`. That will stage that secret in Fly, ready to deploy it:

```
Secrets are staged for the first deployment
```

Now you can go ahead and run `fly deploy` and the build should proceed:

```
...
--> Building image done
==> Pushing image to fly
...
```

You should see the build progress, the healthchecks pass, and a message to confirm the application was successfully deployed.

You have successfully built and deployed your Laravel application on Fly.

### View your application on Fly

Use `fly open` as a shortcut to open the app's URL in your browser. If you are using http, Fly will upgrade it to https.

Use `fly logs` to see the log files.

Use `fly status` to see its details:

```
App
  Name     = your-app-name
  Owner    =
  Version  = 1
  Status   = running
  Hostname = your-app-name.fly.dev

Deployment Status
  ID          = a3c2f40e-bed9-4ce1-923a-9d8ad3183a1c
  Version     = v1
  Status      = successful
  Description = Deployment completed successfully
  Instances   = 1 desired, 1 placed, 1 healthy, 0 unhealthy

Instances
ID      	PROCESS	VERSION	REGION	DESIRED	STATUS 	HEALTH CHECKS     	RESTARTS	CREATED
abcdefgh	app    	1     	lhr   	run    	running	2 total, 2 passing	0       	0h10m ago
```
