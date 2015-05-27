---
layout: post
title:  "Magento Routing"
categories: magento
---

Magento first moves through a series of bootstrap operations before routing is
considered. You can safely skip past the following list, as it is not related
to routing (perhaps with the exception points 1 and 9), also, I'll cover the
below in more detail in a future post.

1. All requests to nonexistent files/directories/links, and anything in `media`,
   get rewritten to `index.php` via the `.htaccess` file
2. `Mage::run()` is called, which does very little
3. `Mage_Core_Model_App::run()`
4. The config and cache are initialized
5. The system cache model is asked to fulfill the request, in vanilla CE this
   will always return false
6. Module config is loaded, DB schema updates are applied
7. The `global` and `events` app areas are loaded
8. Stores are loaded, along with their config, the current store is set
9. The request object is created
10. DB data updates are applied

We have now arrived at a point where we can start the routing process. As with
traditional MVC, this consists of the following objects:

- A single front controller
- One or more routers
- One or more controllers
- A request object
- A response object

The top three objects above end up composing what you could see as a tree:

	|- front controller
		|- router1
			|- controller1
			|- controllerN
		|- routerN
			|- controller1
			|- controllerN

The bottom two objects are representations of a HTTP request and response.

The front controller class name is `Mage_Core_Controller_Varien_Front`, and is
initialised from `Mage_Core_Model_App::_initFrontController()` - all that method
does is instantiate the front controller and call `init()` on it, it is then
the front controllers job to fetch all the routers from config, which are all
defined in XML, in `default/web/routers`. By default, Magento uses 4 routers:

1. admin (`Mage_Core_Controller_Varien_Router_Admin`)
2. frontend (`Mage_Core_Controller_Varien_Router_Standard`)
3. cms (`Mage_Cms_Controller_Router`)
4. default (`Mage_Core_Controller_Varien_Router_Default`)

Controllers are normally direct descendants of either:

* `Mage_Core_Controller_Varien_Action` for frontend controllers
* `Mage_Adminhtml_Controller_Action` for admin controllers

Note that the router list above is ordered on purpose, because ordering is
important for the next step of the process, which is the routing loop. This
loop continues until a router marks the request as `dispatched`, here is an
example flow for a fictitous CMS page, call it http://example.com/cms-page:

1. The front controller calls `match()` on the admin router, which returns
   false as this is not an admin route
2. The frontend router has the same treatment applied to it, again, no match
3. Now for the CMS controller - when `match()` is called it consults the DB
   for a page with an identifier of `cms-page` and finds one, it now alters
   the request object by calling some methods on it:
		   
		$request->setModuleName('cms')
		  ->setControllerName('page')
		  ->setActionName('view')
		  ->setParam('page_id', $pageId);

4. It now returns true, which causes the front controller to restart the
   router `match()` loop (which it will do to a maximum of 100 times),
   which it does because the request object reports that it has not yet
   been dispatched
5. The process is started again from step 1, again, no match on admin, but
   this time to frontend router does match, because the methods the CMS
   router called have set data that allows the frontend router to know
   how to route the request
   
So at this point, the frontend router is in the `match()` method, and as
such has control of the dispatch process, from here it calls `dispatch()`
on the controller that matched the request.

Note that each router could behave differently - though probably never
seen in practice, you could make a router which did not use controllers
and just directly fulfilled the request, so the information below is not
general to all routers (although you will find in practice most will work
very similarly, certainly with Magento the same is true of the admin router).

So now the controller `dispatch()` method is being executed. As with routers,
each controller can do what it wants, but the standard is below:

1. `preDispatch()` is called, which can stop the request going any further
   by marking the request as dispatched, or by setting the `no-dispatch`
   flag on itself
2. Assuming the request is to continue, the actual action gets called, i.e
   `viewAction()` for a CMS page view
3. `postDispatch()` is called

At this point, whatever the controller wished to do has been done, this is
most likely to render a page, but could be something else such as performing
a redirect. From here, control passes back to the router (which does nothing
but return true from `match()`, then back to the front controller, which
quickly proceeds to send the response to the browser.

And that is the whole routing process finished! Routing is quite simple
really, and the class files involved with the process are all fairly small - 
the front controller and standard routers are both less than 500 lines, and
the majority of code in the 100 line action controller class is unrealted
to routing, the most important part is to wrap your head around the fact the
the process is a loop, with potentially multiple routers and controllers
being involved in a single request.



MORE STUFF:

* direct requests, i.e. catalog/product/view/id/5
* Adding modules before and after other modules in a router
* how modules are loaded, maybe? might be too far
* 404

http://alanstorm.com/magento_dispatch_standard_router
