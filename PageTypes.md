# Using custom Page Types in ProcessWire

As ProcessWire users you're probably aware of the fact that core objects like users, roles or languages are just pages. But that fact can actually be missed out quite easily as ProcessWire's api does offer lot's of conveniencies, which are specific to those objects.

```php
// Standard Pages
$page = $pages->get("template=basic_page, title*=Boston");
$page->is("hasRailstation=1");

// Users
$user = $users->get("name=superuser"); // Custom repository
$user->isSuperuser(); // Custom method
```

Now it wouldn't be ProcessWire and it's extendable nature, if you couldn't add such conveniences for yourself. In the following I'll show you exactly how to do that, just be aware, that I'll expect a basic understanding of OOP and class inheritance going forward. 

First we'll look at how to create a custom repository api variable and after that we'll go an see how we can add custom methods to those pages, that our new repository as well as the `$pages` api variable will return.

## When to use this approach

Before we start I want to mention, that this approach is not really beneficial if you just want to add a single method here and there. Also it's not recommended to create new classes for any small group of 3-5 pages. This tools, that we'll be using here, are most useful if you know you'll be working with a type of data extensively in a good bunch of code.  

## Custom repository API variable

Now let's get onto the topic. When looking into the source files of those existing api variables there's one important similarity. They do all extend the same class, which is `PagesType`. This [class](https://github.com/ryancramerdesign/ProcessWire/blob/master/wire/core/PagesType.php) is a baseline implementation of what we're trying to create – just flexible enough, so it just needs some setup to be ready to use.

As an example scenario, which will make it easier for me to explain all the settings, we'll imagine we're building a website, which besides some static content does feature an not so small events section. Now we can already guess that we'll often need to work with event pages. Maybe there will be various places, where we need to select events happening in specified timeframes. We don't want to repeat the selector for such a specific selection in all those places, but rather handle them once and reuse the logic later. 

To do so we'll first add an `events.php` file somewhere in the `site/` folder. We'll use `site/api/events.php` for this tutorial. If you're using an class autoloader like composer (or ProcessWire >3.0) than you can use that to load the file, but we'll just simply include the file via the `site/config.php`.<span id="anchor-include"></span>

```php
// site/config.php
…
include_once( __DIR__ . "api/events.php");
```

And on to the new class file we just created.

```php
<?php
// site/api/events.php

class Events extends PagesType {

	/**
	 * Construct the Events manager for the given parent and template
	 *
	 * @param Template|int|string|array $templates Template object or array of template objects, names or IDs
	 * @param int|Page|array $parents Parent ID or array of parent IDs (may also be Page or array of Page objects)
	 */
	public function __construct($templates = array(), $parents = array()) {
		parent::__construct($templates, $parents);

		// Make sure we always include the event template and events/ parent page
		$this->addTemplates("event");
		$this->addParents($this->pages->get("events/")->id);
	}

	/**
	 * Custom method to find by date range
	 *
	 * @parem int $start timestamp
	 * @parem int $end timestamp
	 * @parem string|null $additionalSelector (optional)
	 * @return PageArray
	 */
	public function findInDateRange($start, $end, $additionalSelector = null){
		// Build selector
		$selector = "enddate>=$start, (enddate<=$end), (startdate<=$end)";
		if($additionalSelector) $selector .= ", " . $additionalSelector;

		// Search only the available events with the selector
		return $this->find($selector);
	}
}
```

That all there is to it for now. With the strong base implementation of `PagesType` we just need to set our new class to always use our custom template and it's parent page and we're done. This new class will from now on only search the pages, which fit our specified criteria (and optionally the ones passed into the constructor).

The second method is for our custom need to search by a timeframe. It's only constructing a selector and then query it's pages for that selector. Notice, that we don't need to specify the template or parent page here anymore. That's already been taken care of.

Now we've a class that does handle all the event management, but we still need another step to actually use it in our templates. We need to instanciate the class and create an api variable to hold it. For that we'll create a `site/init.php` file – if not already existing – and add the following.

```php
<?php
// site/init.php

$events = new Events();
$this->wire('events', $events, true);
```

Now everywhere, where the ProcessWire api variables are present, there will additionally be a `$events` or `wire('events')` variable present and we can simply call our custom find method to retrieve some christmas events.

```php
$from = strtotime('2015-12-01');
$to = strtotime('2015-12-24');
$christmasEvents = $events->findInDateRange($from, $to, "christmas=1");
```

## Custom methods for events

In this second part of the tutorial we'll add custom methods to the event instances we'll be getting from our new `$events` api variable. For our example we'll be implementing a check if an event does start at the current day, which might be something often required throughout the site.

There are two ways of extending ProcessWire objects – Hooks and inheritance. Technically we could've used hooks for the above setup as well, but I'd not recommend that. But for adding methods to pages it's actually a plausable alternative to inheritance. Therefore I'll include a short example on how to hook methods in here as well.

### Hooking

```php
// site/init.php
…
$wire->addHook("Page::startsToday", function(HookEvent $event){
	$page = $event->object;
	if($page->template != 'event') return;

	return $event->return = date("Y-m-d", $page->getUnformatted("startdate")) == date("Y-m-d");
});
```

As you can see, we're just adding a new method to all pages, but if such a page is not an event, we'll just return `null`. That's a fine way to go as long as the number or complexity of those methods doesn't grow much more. Imagine fetching multiple parameter values of a custom method from the injected `HookEvent` and maybe doing all the type checking manually. That's a good bulk of boilerplate code just to get to match what a class method does do without even looking at the function body. Also for each new method, we would add to events, we would need to implement the check for the correct template, which can easily lead to lot's of code duplication.

### Inheritance

The second option we have is extending the `Page` class like we did it above with the `PagesType` class. For that we'll create another file `site/api/event.php` and include it like the one [previously](#anchor-include). 

```php
<?php
// site/api/event.php

class Event extends Page{

	/**
	 * Create a new Event page in memory. 
	 *
	 * @param Template $tpl Template object this page should use.
	 */
	public function __construct(Template $tpl = null) {
		if(is_null($tpl)) $tpl = $this->templates->get('event'); 
		if(!$this->parent_id) $this->set('parent_id', $this->pages->get("events/")->id); 
		parent::__construct($tpl); 
	}

	/**
	 * Does the event start today
	 * 
	 * @return bool
	 */
	public function startsToday(){
		return date("Y-m-d", $this->getUnformatted("startdate")) == date("Y-m-d");
	}
}
```

As you can see, this is quite similar to the other class above. We make sure, that even without the constructor paramter we'll instanciate the class for the correct parent page and template and we've added the same method we added by the hook before, just in a much more readable fashion.

With that class we can now do things like that:

```php
$event = new Event();
$event->startdate = time();
$event->startsToday();
```

Notice, that we again don't need to care about any template or parent setting. It's already specified in the class itself. But by now the class will only be used if we instanciate a new event. To make ProcessWire use this class we'll need to go and enable the `advanced` mode in the `site/config.php` and then edit the `event` template in the backend. Under the system tab we set `Event` as the class to be used for that template, so all event pages we'll receive from any api variable are actually of our new `Event` class.

Additionally we'll add the following line in the constructor of our `Events` class, which is just an extra enforcement, but it also makes our custom repository class more predictable by specifically stateing that it should return objects of that class.

```php
// site/api/events.php
…
	public function __construct($templates = array(), $parents = array()) {
		[…]

		$this->setPageClass('Event');
	}
…
```

## Finish up.

Now you've seen that – while it's not that complicated – there's actually quite a bit to setup required. That's the reason why I do not recommended to use this in a holy grail fashion, but rather if you really need. It doesn't make sense to create those classes for types of data you don't need often enought throughout a whole site.