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

Now it wouldn't be ProcessWire and it's extentible nature, if you couldn't add such conveniences for yourself. In the following I'll show you exactly how to do that. Just be aware, that I'll expect a basic understanding of OOP and class inheritance going forward. At first we'll look at how to create a custom repository api variable and after that we'll go an see how we can add custom methods to those pages, that the repository as well as the `$pages` api variable does return.

## When to use this approach

…

## Custom repository API variable

When looking into the source files for those existing api variables there's one important similarity, which is the class they both extend, which is `PagesType`. The [class](https://github.com/ryancramerdesign/ProcessWire/blob/master/wire/core/PagesType.php) is a baseline implementation of what we're trying to create, just flexible enough, so we just need to setup some of the properties and be good to go.

Now with the dry theory done we'll need an example scenario that'll make it easier for me to explain all the settings. Imagine we're building a website, which besides some static content does feature an events section. Now we can already guess that we'll often need to select events based on a different timeframes. We don't want to repeat the selector for that specific selection in all those cases, but rather handle them once and reuse the logic later. 

First we'll add `events.php` somewhere in the `site/` folder. We'll use `site/api/events.php` for this tutorial. If you're using an autoloader like composer (or ProcessWire >3.0) than you can use it to load the file, but we'll just simply include the file via the `config.php`.<span id="anchor-include"></span>

```php
// site/config.php
…
include_once( __DIR__ . "api/events.php");
```

And on to the new file:

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
	 * @parem string|null $selector
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

With that we've a class that does handle all the event management, but we still need another step to actually use this in our templates. We need to instanciate the class and create an api variable to hold it. For that we'll create a `site/init.php` file – if not already existing – and add the following.

```php
<?php
// site/init.php

$events = new Events();
$this->wire('events', $events, true);
```

Now everywhere, where the ProcessWire api variables are present, there will be a `$events` or `wire('events')` variable present and we can simply call our custom find method to retrieve some christmas events.

```php
$christmasEvents = $events->findInDateRange(strtotime('2015-12-01'), strtotime('2015-12-24'), "christmas=1");
```

## Custom methods for pages

The second part of the tutorial is about adding custom methods to classes. There are two ways of doing that – Hooking and Inheritance. We'll shortly take a look into hooks, but as we've already a custom repository we'll cover the inheritance way a tad more detailed.

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

This is all fine – especially for few smaller methods – but it also can cause a ton of builderplate for example for methods with lot's of parameters.

### Inheritance

The second option is extending the `Page` class like we did above for the `PagesType` class. We'll create `site/api/event.php` and include it like the `site/api/events.php` [previously](#anchor-include). 

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

Now we've our custom event page, that will automatically set itself up with the correct parent/template and hold our custom methods. 

```php
$event = new Event();
$event->startdate = time();
$event->startsToday();
```

To finish this off we'll go and enable the `advanced` mode in the `site/config.php` and edit the `event` template. In the system tab we set `Event` as the class to be used for that template, so all pages you receive from the `$pages` api variable are actually of our new Event class. Also we'll add a line in the constructor of our `Events` class, which is just an additional enforcement, but makes our custom repository class more predictable. 

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

As you've seen this is quite a lot of stuff to do for just adding two methods. That's why I disclosed this as a tool for bigger sites, where ProcessWire does not only render a website, but is used in the framework sense. 