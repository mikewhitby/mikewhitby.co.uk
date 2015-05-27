---
layout: post
title:  "Symfony - From Zero to Hello World"
categories: symfony
---

As I've been working with Magento for a long time I thought it was time to get
started with another framework to broaden my horizons, and Symfony seems like a
pretty good choice to get in to right now, what with Oro and Sylius using the
framework, so I apologise if this post is a little bit Magento flavoured, i.e.
"Magento does it this way, but Symfony does not".

I can't be bothered to type "2" all the time, so I'll just call it Symfony.

This blog post (And any that follow it, we'll see how I feel about that later)
will focus around a personal money tracking application that I'll make, lets
call it cabbage, why not?


## RTFM!

I'm following the [Symfony book][1] to get me started. The first thing that
strikes me is how thoroughly documented Symfony is - this is light years away
from when I was learning Magento! I had a brief read of the first couple of
chapters - chapter one reinforces the fact that Symfony (and indeed all PHP
frameworks) are there for (usually) one person; to answer a HTTP request and
send a HTTP response. This might be obvious to some people but I think this
simplicity of focus is often lost in todays plethora of technologies involved in
any one applications stack. The second chapter is quite nice as well too
actually - it highlights why you would use a framework like Symfony by comparing
a typical application, implemented in both Symfony and plain ol' PHP - quite a
refreshing reminder of the pain that caused you to shift to MVC a long time ago,
a pain which may have now faded in your mind!


## Installing Symfony

The book wanted me to use the [Symfony Installer][2], hmm, not the biggest fan
of having to use tools like this simply to install a project, what is wrong with
git clone or composer? We'll see in time if this command line tool provides me
with any real benefits, right now I'm a bit sceptical of it, anyway, here we go:

    curl -LsS http://symfony.com/installer -o ~/bin/symfony
    chmod +x ~/bin/symfony
    symfony new cabbage

Well that worked nicely, and the output tells me I can run a webserver (using
PHP 5.5's built in webserver), so lets try that...

    cd cabbage
    php app/console server:run

Well that was the easiest project setup I've ever done! Whilst it would seem a
good idea to get a VM running (because in order to do this, I'm going to have to
install MySQL on my mac) but I plan to use this webserver initially, until I'm
serious about the project, which is exactly what it is for I suppose, great!

Finally, lets commit the project. I find git invaluable when learning, because I
often change a file and forget about it, or perhaps I don't work on the project
for a few days, but git refreshes my memory and gets me right back up to speed.
The first thing I would do with Magento is to spend ages figuring out where the
hell I put my stock `.gitignore`, then to spend a while looking at ones
available online, use them, figure they are no good, whittle them down, and
finally end up with something reasonable, but with Symfony, what's this? It
looks like there is already a `.gitignore` present, awesome! So it really is a
case of:

    git add -A
    git ci -m "Initial commit"

So, now I'm ready to do development - seriously impressed with Symfony so far.


## Removing The Demo Bundle

Oh ok, not quite - seems we have a module present called `AcmeDemoBundle`, and
I have a phobia of leaving any demo stuff around - I like to start from a clean
slate, so this has to go - if I need to look at code from it, I'll create a new
Symfony project with it in.

By the way, for those that don't know, a bundle is the name Symfony gives to a
self-contained collection of code/config/etc, basically think "module", or
"extension" etc, well, Symfony calls it a bundle. The book says that there is no
point having bundles that rely on one another, and that in this case you should
use one bundle - this sounds a bit idealist I think, I'm pretty sure that in
practice this won't hold up. Apparently I should be using AppBundle only for my
cabbage code, I suppose this wouldn't be true if you were making a framework
like Oro, but for me it sounds reasonable, although a far cry from my Magento
roots where you would many small modules, each serving a single purpose - I'll
do as told though, and see how it goes.

Anyway, back to removing the demo bundle, looks like Symfony puts all of "my
stuff" in to `src`, and I can see `Acme\DemoBundle` in there, so I'll just
remote that i suppose...

Ah, no. I refresh the page and am greeted with an error page:

> ClassNotFoundException in AppKernel.php line 24:
> Attempted to load class "AcmeDemoBundle" from namespace "Acme\DemoBundle".
> Did you forget a "use" statement for another namespace?

Hm, ok, so it looks like the bundle was registered in `app\AppKernel.php`, I had to
go in and remove line 24 from the file. Lets try again...ah no, another error:

> Bundle "AcmeDemoBundle" does not exist or it is not enabled. Maybe you
> forgot to add it in the registerBundles() method of your AppKernel.php file?
> in @AcmeDemoBundle/Resources/config/routing.yml (which is being imported
> from "/Users/mike/Projects/cabbage2/app/config/routing_dev.yml"). Make sure
> the "AcmeDemoBundle" bundle is correctly registered and loaded in the
> application kernel class. If the bundle is registered, make sure the bundle
> path "@AcmeDemoBundle/Resources/config/routing.yml" is not empty.

So, looks like the error is to do with `routing_dev.yml`, and sure enough, we
have lines in the routing file for our bundle:

    # AcmeDemoBundle routes (to be removed)
    _acme_demo:
        resource: "@AcmeDemoBundle/Resources/config/routing.yml"

Having removed these, I now get a error saying there is no route, which sounds
fine, looks like we've removed the bundle. This is the first part of Symfony I
do not like though - I think I should be able to delete a directory and the
bundle should be gone, why do I have to edit two global files? To me, this seems
quite strange, how would I download a bundle and have it automatically
activated? In Magento I can disable a module by removing one XML file, which
means that conversely, I can download and activate a module without altering any
core files, so this concept seems quite alien to me.


## Hello World

Right, we're finally ready to get going! As I'll be making a personal finance
tracker I will want to be able to list, add and edit accounts, so I'll make a
conroller which will serve a request of `/acounts` - right now I'll just make it
serve a "Hello, world!" page.

My Magento instincts tell me this is the time to create a module, ready for a
controller, however I will follow the book and use the AppBundle for my code.
First of all, the book is asking me to declare my routing configuration, though
having had a look in `app/config/routing.yml` it looks like the config for
`AppBundle` is already present:

    app:
        resource: "@AppBundle/Controller/"
        type:     annotation

This yaml isn't the same as the book tells me to make, which is this:

    acme_website:
        resource: "@AcmeDemoBundle/Resources/config/routing.yml"
        prefix:   /

I remember from chapters 1 and 2 that the first example instructs Symfony to
map controllers and methods to routes via @annotations, whereas the second
uses the `routing.yml` in the bundle to do the mapping. It seems to me that
annotations are the cleanest way to do what I want, so I'll use those, so I
don't need a `routing.yml` in my bundle, so lets make the controller...

Oh, we already have one! A file exists called
`src/AppBundle/Controller/DefaultController.php`, which has an `indexAction()`,
with annotations that map `/app/example` to itself, surely enough, visiting that
URI results in "Homepage." being displayed


[1]: http://symfony.com/doc/current/book/index.html
[2]: http://symfony.com/doc/current/book/installation.html
