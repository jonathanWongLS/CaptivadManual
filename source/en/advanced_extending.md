<!--toc=advanced-->
# Extending the CMS

This section will discuss custom extensions to the CMS, but is also useful
reading for a developer wanting the extend the core CMS and contribute that
change back to the project.

It will discuss the CMS architecture and how that can be leveraged to make
changes.

Minor adjustments to the visual style of the CMS or provision of new Modules
is addressed in the Themes and Modules sections respectively.

Integration with 3rd party systems should always be achieved using the API
rather than direct modification of the CMS.

<white>
Please contact your provider for further development details.
</white>

<nonwhite>

## Architecture

The CMS is a PHP web application which uses the Slim2 framework. It's core
principle is that the majority of routes should be accessible via a RESTful
interface.

If Slim2 is entirely new then it is advisable to break out of this documentation
and review the [Slim2 documentation](http://docs.slimframework.com/) to
understand how that works. Slim3 has since been released and migrating to
it is a development item for 1.9 Release. At that time it will be necessary
to update any integration to suite Slim3 - a migrating guide will be published.

The CMS has two main entry points:

 - `/web/index.php`
 - `/web/api/index.php`

Requests to each point are routed to the same end controller - for example:

 - `GET http://xibo/display`
 - `GET http://xibo/api/display`

Both end up at `Display::grid()` - the CMS framework knows how to format each
response accordingly, because of the Middleware that has been wrapped around
each entry point.

Routes accessed via the API entry point will not have Twig templates available
for formatting the response and in these cases the response will be JSON
formatted. The entry point can be tested by calling `$app->getName()`.

### Middleware

Middleware surrounds the Slim application like an onion and is executed
before and after the main request in LIFO order. Additional Middleware can be
provided by adding to the `$middleware` array in `/web/settings.php`.

The Middleware added by the core application depends on the entry point that
has been used. For example, the web/api entry points use a different
authentication Middleware (one cms-auth and the other oauth2).

Custom Middleware is added to all entry points.

### Routing

Routes are added using short hand notation, for example:

```
$app->get('/display', '\Xibo\Controller\Display:grid')->name('display.search');
```

When routes are matched the Controller is looked up in the DI container.
Therefore controllers should always be added to the DI container.

### Dependency Injection

The CMS strictly injects all dependencies into the constructor of each object
it needs. Controller and Factories (these are actually Repositories) are
setup in Slim's DI container by the `State` Middleware.

To provide additional controllers, these should be registered to the
`Slim::container` in Middleware.

### Database access

The database is MySQL and is accessible through a `PdoStorageService`
configured in the `Storage` Middleware.

### Application State

The framework provides an `ApplicationState` object which can be used to
conditionally format a response. It is not necessary to use this object if you
would rather format your own response.

### Twig templates

When using the `/web/index.php` entry point it is advisable but not necessary
to render output to the browser using the Twig template library. This is
included in the CMS code and has helper methods in the `ApplicationState`
object. This should be done for any page which should output HTML.

Themes should be used to provide additional templates for rendering.

A Controller should use the `ApplicationState` object and set the following
properties:

 - `ApplicationState::template` - set to the name of the template without it's
 extension.
 - `ApplicationState::setData()` - an array of data to provide to the template

### Logging

While developing it is advisable to put the CMS installation into Test mode so that full
logging output is available in the event of any errors. More information is available 
[here](advanced_logging.html).


## Examples

### Wiring up a new theme link

You've developed a custom theme which exposes a new link on the navigation menu
(or anywhere else). 

**We recommend you use a theme for custom modifications so that they are not overwritten
during an upgrade**. 

This link should take some action provided by a Custom Route/Controller.

Your link looks like:

```
{% if currentUser.routeViewable("/myapp") %}
<li class="sidebar-list"><a href="{{ urlFor("test.view") }}">{% trans "Test" %}</a></li>
{% endif %}
```

The `urlFor` method will look to connect to a named route - `test.view` - and
this route will need to be provided.

Middleware is used to wire up the new route - create a Middleware file in the
`/custom` folder.

```
<?php
namespace Xibo\Custom;

use Slim\Middleware;

/**
 * Class MyMiddleware
 * @package Xibo\custom
 *
 * Included by instantiation in `settings.php`
 */
class MyMiddleware extends Middleware
{
    public function call()
    {
        $app = $this->getApplication();

        // Register some new routes
        $app->get('/myapp/test', '\Xibo\Custom\MyController:testView')->setName('test.view');

        // Register a new controller with DI
        // This Controller uses the CMS standard set of dependencies. Your controller can
        // use any dependencies it requires.
        // If you want to inject Factory objects, be wary of circular references.
        $app->container->singleton('\Xibo\Custom\MyController', function($container) {
            return new \Xibo\Custom\MyController(
                $container->logService,
                $container->sanitizerService,
                $container->state,
                $container->user,
                $container->helpService,
                $container->dateService,
                $container->configService
            );
        });

        // Next middleware
        $this->next->call();
    }
}
```

The URL chosen will be automatically checked for permissions by the CMS
Middleware. If you want the route to be accessible by users other than super
admins, you should add `myapp` as a page record in the `pages` database table.

The Controller should be provided also:

```
<?php
namespace Xibo\Custom;

use Xibo\Controller\Base;
use Xibo\Service\ConfigServiceInterface;
use Xibo\Service\DateServiceInterface;
use Xibo\Service\LogServiceInterface;
use Xibo\Service\SanitizerServiceInterface;

/**
 * Class MyController
 * @package Xibo\Custom
 */
class MyController extends Base
{
    /**
     * Set common dependencies.
     * @param LogServiceInterface $log
     * @param SanitizerServiceInterface $sanitizerService
     * @param \Xibo\Helper\ApplicationState $state
     * @param \Xibo\Entity\User $user
     * @param \Xibo\Service\HelpServiceInterface $help
     * @param DateServiceInterface $date
     * @param ConfigServiceInterface $config
     */
    public function __construct($log, $sanitizerService, $state, $user, $help, $date, $config)
    {
        $this->setCommonDependencies($log, $sanitizerService, $state, $user, $help, $date, $config);
    }

    /**
     * Display Page for Test View
     */
    public function testView()
    {
        // Call to render the template
        // This assumes that "twig-template-name-without-extension.twig" exists
        // in the active theme (i.e. in the view path of the active theme).
        $this->getState()->template = 'twig-template-name-without-extension';
        $this->getState()->setData([]); /* Data array to provide to the template */
    }
}
```

The Middleware needs to be wired up in the `settings.php` file:

```
$middleware = [new \Xibo\Custom\MyMiddleware()];
```

With this code added, visiting the CMS causes the new Middleware to be called,
which registers your new Controller and Route. If the link visited matches
your route, the Controller Method is executed and its output rendered.

This Controller example uses the `ApplicationState` to render a Twig template. The
template would need to be available in the view path of the active theme. If you
are using a custom theme this is `/web/theme/custom/themeName/views` (unless you have
specified an alternative path in your theme config).

Create a file in your theme named `twig-template-name-without-extension.twig` for the simplest 
example of a Twig template.
 
```
{% extends "authed.twig" %}
{% import "inline.twig" as inline %}

{% block pageContent %}
    <div class="widget">
        <div class="widget-title">{% trans "Page Name" %}</div>
        <div class="widget-body">
            
        </div>
    </div>
{% endblock %}
```

Alternatively you can directly modify the browser output within the controller. To do this
you need to inform the `ApplicationState` object that it should not output anything. For
example:

```
$this->setNoOutput(true);
echo 'This is my output to the browser';
```

</nonwhite>
