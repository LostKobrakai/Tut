# Testing with ProcessWire 3.0 and MySQL 5.6

This is more a loose task-list than a tutorial, but maybe it's interesting to others:

First we'll begin with some basic steps: 

- Prepare a server running MySQL 5.6 and composer
- Install the current `devns` version of ProcessWire. Use InnoDB for the database engine.
- Make sure the tables are indeed using InnoDB (mine were not).

- Go into `/site/` and run `composer require --dev crysalead/kahlan`
- Create a new subfolder `/site/spec/` and place a file `TransactionSpec.php` in it.
- Edit the file and paste in the following:  
```php
<?php

describe("InnoDB transaction functionality", function(){

	before(function()
		include_once("../index.php");
		foreach($wire->wire('all')->getArray() as $api => $data){
			if($api == "process") continue;
			$this->$api = $data; // Make api variables available in tests
		}
	});

	it("should remove created pages", function(){
		$this->database->beginTransaction();
		expect($this->database->inTransaction())->toBe(true);

		$p = new ProcessWire\Page();
		$p->parent = 1;
		$p->template = "basic-page";
		$p->name = "test";
		$p->title = "Test";
		$p->save();

		expect($this->database->rollBack())->toBe(true);
		expect($this->pages->get("name=test, parent=1"))->toBeAnInstanceOf('ProcessWire\NullPage');
	});


	it("should reset changes made to fields", function(){
		$this->database->beginTransaction();
		expect($this->database->inTransaction())->toBe(true);

		$p = $this->pages->get("/");
		$p->title = "Test";
		$p->save();

		expect($this->database->rollBack())->toBe(true);
		expect($this->pages->get("/")->title)->toBe('Home');
	});

});
```

- Edit the `/site/composer.json` and add the following:  
```json
{
	...,
	"autoload": {
		"files": ["../wire/core/ProcessWire.php"]
	}
}
```

- run `composer dump-autoload`
- run `vendor/bin/kahlan` and hope all tests go well

Mix this with some database seeding and some way to make sure the test does always run on the same database state will make testing your ProcessWire Project a breeze. 