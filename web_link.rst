Asset Preloading and Resource Hints with HTTP/2 and WebLink
===========================================================

Symfony provides native support (via the `WebLink`_ component)
for managing ``Link`` HTTP headers, which are the key to improve the application
performance when using HTTP/2 and preloading capabilities of modern web browsers.

``Link`` headers are used in `HTTP/2 Server Push`_ and W3C's `Resource Hints`_
to push resources (e.g. CSS and JavaScript files) to clients before they even
know that they need them. WebLink also enables other optimizations that work
with HTTP 1.x:

* Asking the browser to fetch or to render another web page in the background;
* Making early DNS lookups, TCP handshakes or TLS negotiations.

Something important to consider is that all these HTTP/2 features require a
secure HTTPS connection, even when working on your local machine. The main web
servers (Apache, nginx, Caddy, etc.) support this, but you can also use the
`Docker installer and runtime for Symfony`_ created by Kévin Dunglas, from the
Symfony community.

Preloading Assets
-----------------

Imagine that your application includes a web page like this:

.. code-block:: html

    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="UTF-8">
        <title>My Application</title>
        <link rel="stylesheet" href="/app.css">
    </head>
    <body>
        <main role="main" class="container">
            <!-- ... -->
        </main>
    </body>
    </html>

Following the traditional HTTP workflow, when this page is served browsers will
make one request for the HTML page and another request for the linked CSS file.
However, thanks to HTTP/2 your application can start sending the CSS file
contents even before browsers request them.

To do that, first install the WebLink component:

.. code-block:: terminal

    $ composer require symfony/web-link

Now, update the template to use the ``preload()`` Twig function provided by
WebLink. The `"as" attribute`_ is mandatory because browsers need it to apply
correct prioritization and the content security policy:

.. code-block:: html+twig

    <head>
        <!-- ... -->
        <link rel="preload" href="{{ preload('/app.css', { as: 'style' }) }}">
    </head>

If you reload the page, the perceived performance will improve because the
server responded with both the HTML page and the CSS file when the browser only
requested the HTML page.

.. note::

    You can preload an asset by wrapping it with the ``preload()`` function:

    .. code-block:: html+twig

        <head>
            <!-- ... -->
            <link rel="preload" href="{{ preload(asset('build/app.css')) }}">
        </head>

Additionally, according to `the Priority Hints specification`_, you can signal
the priority of the resource to download using the ``importance`` attribute:

.. code-block:: html+twig

    <head>
        <!-- ... -->
        <link rel="preload" href="{{ preload('/app.css', { as: 'style', importance: 'low' }) }}">
    </head>

How does it work?
~~~~~~~~~~~~~~~~~

The WebLink component manages the ``Link`` HTTP headers added to the response.
When using the ``preload()`` function in the previous example, the following
header was added to the response: ``Link </app.css>; rel="preload"; as="style"``
According to `the Preload specification`_, when an HTTP/2 server detects that
the original (HTTP 1.x) response contains this HTTP header, it will
automatically trigger a push for the related file in the same HTTP/2 connection.

Popular proxy services and CDNs including `Cloudflare`_, `Fastly`_ and `Akamai`_
also leverage this feature. It means that you can push resources to clients and
improve performance of your applications in production right now.

If you want to prevent the push but let the browser preload the resource by
issuing an early separate HTTP request, use the ``nopush`` option:

.. code-block:: html+twig

    <head>
        <!-- ... -->
        <link rel="preload" href="{{ preload('/app.css', { as: 'style', nopush: true }) }}">
    </head>

Resource Hints
--------------

`Resource Hints`_ are used by applications to help browsers when deciding which
resources should be downloaded, preprocessed or connected to first.

The WebLink component provides the following Twig functions to send those hints:

* ``dns_prefetch()``: "indicates an origin (e.g. ``https://foo.cloudfront.net``)
  that will be used to fetch required resources, and that the user agent should
  resolve as early as possible".
* ``preconnect()``: "indicates an origin (e.g. ``https://www.google-analytics.com``)
  that will be used to fetch required resources. Initiating an early connection,
  which includes the DNS lookup, TCP handshake, and optional TLS negotiation, allows
  the user agent to mask the high latency costs of establishing a connection".
* ``prefetch()``: "identifies a resource that might be required by the next
  navigation, and that the user agent *should* fetch, such that the user agent
  can deliver a faster response once the resource is requested in the future".
* ``prerender()``: "identifies a resource that might be required by the next
  navigation, and that the user agent *should* fetch and execute, such that the
  user agent can deliver a faster response once the resource is requested later".

The component also supports sending HTTP links not related to performance and
any link implementing the `PSR-13`_ standard. For instance, any
`link defined in the HTML specification`_:

.. code-block:: html+twig

    <head>
        <!-- ... -->
        <link rel="alternate" href="{{ link('/index.jsonld', 'alternate') }}">
        <link rel="preload" href="{{ preload('/app.css', { as: 'style', nopush: true }) }}">
    </head>

The previous snippet will result in this HTTP header being sent to the client:
``Link: </index.jsonld>; rel="alternate",</app.css>; rel="preload"; nopush``

You can also add links to the HTTP response directly from controllers and services::

    // src/Controller/BlogController.php
    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\WebLink\GenericLinkProvider;
    use Symfony\Component\WebLink\Link;

    class BlogController extends AbstractController
    {
        public function index(Request $request): Response
        {
            // using the addLink() shortcut provided by AbstractController
            $this->addLink($request, new Link('preload', '/app.css'));

            // alternative if you don't want to use the addLink() shortcut
            $linkProvider = $request->attributes->get('_links', new GenericLinkProvider());
            $request->attributes->set('_links', $linkProvider->withLink(
                (new Link('preload', '/app.css'))->withAttribute('as', 'style')
            ));

            return $this->render('...');
        }
    }

.. _`WebLink`: https://github.com/symfony/web-link
.. _`HTTP/2 Server Push`: https://tools.ietf.org/html/rfc7540#section-8.2
.. _`Resource Hints`: https://www.w3.org/TR/resource-hints/
.. _`Docker installer and runtime for Symfony`: https://github.com/dunglas/symfony-docker
.. _`"as" attribute`: https://w3c.github.io/preload/#as-attribute
.. _`the Priority Hints specification`: https://wicg.github.io/priority-hints/
.. _`the Preload specification`: https://www.w3.org/TR/preload/#server-push-http-2
.. _`Cloudflare`: https://blog.cloudflare.com/announcing-support-for-http-2-server-push-2/
.. _`Fastly`: https://docs.fastly.com/en/guides/http2-server-push
.. _`Akamai`: https://http2.akamai.com/
.. _`link defined in the HTML specification`: https://html.spec.whatwg.org/dev/links.html#linkTypes
.. _`PSR-13`: https://www.php-fig.org/psr/psr-13/
