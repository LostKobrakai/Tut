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

Now it wouldn't be ProcessWire and its extendable nature, if you couldn't add such conveniences for yourself. In the following I'll show you exactly how to do that, just be aware, that I'll expect a basic understanding of OOP and class inheritance going forward. 

First we'll look at how to create a custom repository api variable and after that we'll go an see how we can add custom methods to those pages, that our new repository as well as the `$pages` api variable will return.

## When to use this approach

Before we start I want to mention, that this approach is not really beneficial if you just want to add a single method here and there. Also it's not recommended to create a new repository classes for any small group of 3-5 pages. These classes, that we'll be creating here, are most useful if you know you'll be working with the type of data extensively throughout your site. Otherwise it may not be worth the effort.

## Custom repository API variable

Now let's get onto the topic. When looking into the source files of those existing api variables there's one important similarity. They do all extend the same class, which is `PagesType`. This [class](https://github.com/ryancramerdesign/ProcessWire/blob/master/wire/core/PagesType.php) is a baseline implementation of what we're trying to create – just flexible enough, so it just needs some setup to be ready to use.

As an example scenario, which will make it easier for me to explain all the settings, we'll imagine we're building a website, which besides some static content does feature an not so small events section. Now we can already guess that we'll often need to work with event pages. Maybe there will be various places, where we need to select events happening in specified timeframes. We don't want to repeat the selector for such a specific selection in all those places, but rather handle them once and reuse the logic later. 

To do so we'll first add an `events.php` file somewhere in the `site/` folder. We'll use `site/api/events.php` for this tutorial. If you're using an class autoloader like [composer](https://getcomposer.org/) (or [ProcessWire 3](https://processwire.com/blog/posts/processwire-2.6.20-and-surprise-processwire-3.0-alpha-1/#what-benefits-will-using-namespaces-bring)) than you can use that to load the file, but we'll just simply include the file via the `site/config.php`.<span id="anchor-include"></span>

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

		// Make sure we always include the event template and /events/ parent page
		$this->addTemplates("event");
		$this->addParents($this->pages->get("/events/")->id);
	}

	/**
	 * Custom method to find by date range
	 *
	 * @param int $start timestamp
	 * @param int $end timestamp
	 * @param string|null $additionalSelector (optional)
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

That all there is to it for now. With the strong base implementation of `PagesType` we just need to set our new class to always use our custom template and its parent page and we're done. This new class will from now on only search the pages, which fit our specified criteria (and optionally the ones passed into the constructor).

The second method is for our custom need to search by a timeframe. It's only building up a selector and then query its pages for that selector. Notice, that we don't need to specify the template or parent page here anymore. That's already been taken care of.

Now we've a class that does handle all the event management, but we still need another step to actually use it in our templates. We need to instanciate the class and create an api variable to hold it. For that we'll create a `site/init.php` file – if not already existing – and add the following.

```php
<?php
// site/init.php

$events = new Events();
$this->wire('events', $events, true);
```

Now everywhere, where the ProcessWire api variables are present, there will additionally be a `$events` or `wire('events')` variable present and we can simply call our custom find method to retrieve some christmas events.

```php
<?php
// site/templates/someTemplate.php

$from = strtotime('2015-12-01');
$to = strtotime('2015-12-24');
$christmasEvents = $events->findInDateRange($from, $to, "christmas=1");

echo "<ul>";
echo $christmasEvents->implode("<li><a href='{url}'>{title}</a></li>");
echo "</ul>";
```

## Custom methods for events

In this second part of the tutorial we'll add custom methods to the event instances we'll be getting from our new `$events` api variable. For our example we'll be implementing a check if an event does start at the current day, which might be something often required throughout the site.

There are two ways of extending ProcessWire objects – Hooks and inheritance. Technically we could've used hooks for the above setup as well, but I'd not recommend that for reasons I'll explain at the end of the tutorial. But for adding few methods to pages hooking is actually a plausable alternative to inheritance. Therefore I'll include a short example on how to hook methods in here as well.

### Hooking

```php
// site/init.php
…
$wire->addHook("Page(template=event)::startsToday", function(HookEvent $event){
	$page = $event->object;
	return $event->return = date("Y-m-d", $page->getUnformatted("startdate")) == date("Y-m-d");
});
```

We're using a [conditional hook](https://processwire.com/blog/posts/new-ajax-driven-inputs-conditional-hooks-template-family-settings-and-more/#new-conditional-hooks) here. This means ProcessWire will make it look like the method does not exist on a page, which isn't an event, and return a result just in the case it is one. This is the way I'd recommend for one-of additions.

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
		if(!$this->parent_id) $this->set('parent_id', $this->pages->get("/events/")->id); 
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

As you can see, this is quite similar to the other class above. We make sure, that even without the constructor parameter we'll instanciate the class for the correct parent page and template and we've added the same method we added by the hook before, just in a much more readable fashion.

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

## Why not use hooks

Before I end here I want to cover the reasons why I sometimes favor inheritance over hooks. Hooks are great, but have some backdraws especially for more complex systems, which benefit most from those classes we've created.

The syntax of hooks, while being quite easy to use, is still harder to reason about when glancing over it and even more if you're not familiar with the codebase. Imagine a method with three or even more parameters and you need to first retrieve all of them from the injected `HookEvent` object and test them against `null` to be sure they're even provided. If you need default values, you need to create the logic manually as well.

Also as hooks are just runtime replacements for real methods they don't have features like visibility properties (`public`, `protected` and `private`) or type hinting. While really a lot of functions can already be hooked for example the constructors cannot and we did change the constructor logic in both of our classes.

### Interfaces

If you're building a larger scale application in ProcessWire there's also a last reason to not use hooking. It's the usage of interfaces. OOP principles state that you shouldn't depend on concrete class implementations, but rather on abstract interfaces. Staying with the events example from above you may want to create a class, which shall remind attending people about the event a few days in advance. You might want to type-hint the class to only receive events, which works short term, but maybe later you also want to remind people of pending meetings as well. Therefore it's better to accept an interface `RemindableInterface` instead of just the class `Event`. The only way for a `Page` object to adhere to such an interface is by creating a class like we did above. Hooks could only add the method, but not add the interface.

```php
<?php
// site/api/event.php

class Event extends Page implements RemindableInterface{

	…

	/**
	 * Event implementation of the interface
	 *
	 * @return array List of emails and optional names of people to be reminded
	 */
	public function getEmailsToBeReminded() {
		return $this->registered_people->explode(function($attendee){
			return array(
				"name" => $attendee->fullname,
				"email" => $attendee->email
			);
		});
	}
}
```

## Finish up.

I hope you enjoyed this excursion into some more advanced ProcessWire usage and you learned something new. Maybe this can encourage you to explore the core files of ProcessWire on your own. It can be exciting to not use a magic box anymore, but understand a bit about what's actually happening.
