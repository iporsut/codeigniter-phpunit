codeigniter-phpunit
===================

This project is a simple hack to make CodeIgniter 2.1.x work seamlessly with PHPUnit 3.7. It aims to provide a way to use PHPUnit standard methodologies for automating tests with CodeIgniter framework, which is notoriously test-unfriendly.

Start Testing
-------------

The files provided here can just be overwritten on top of an existing, vanilla CI application. PHPUnit's `phpunit.xml` will sit besides your application's `index.php`, and hacked CI's *system/core* files should overwrite the vanilla CI ones.

After that, just put your own tests in *application/tests* folder. A `CITest.php` file is provided as an util class, to be used instead of *PHPUnit_Framework_TestCase*, but you can use PHPUnit's standard classes as well.

As an example, a unit test for CI's Email helper would be as follows:

```
<?php

class EmailHelperTest extends PHPUnit_Framework_TestCase
{
    private $CI;
	
	public static function setUpBeforeClass()
	{
		$CI =& get_instance();
		$CI->load->helper('email');
	}
	
	public function testEmailValidation()
	{
		$this->assertTrue(valid_email('test@test.com'));
		$this->assertFalse(valid_email('test#testcom'));
	}
}

?>
```

Just use PHPUnit's command line tool at your application root directory:

`$> phpunit`

How it Works
------------

### PHPUnit Config XML ###

The idea is to use CodeIgniter's own bootstrap file to bootstrap PHPUnit tests, as PHPUnit XML config file allows. So, we'll let CI start its framework as usual, complete with configuration and auto-loading as your application wants. We do this by using a base `phpunit.xml`:

```
<?xml version="1.0" encoding="UTF-8" ?>
<phpunit bootstrap="application/tests/bootstrap.php">
    <testsuites>
		<testsuite name="TestSuite">
			<directory>application/tests</directory>
		</testsuite>
	</testsuites>
	<php>
		<const name="PHPUNIT_TEST" value="1" />
		<server name="REMOTE_ADDR" value="0.0.0.0" />
		<var name="CFG" value="1" />
	</php>
	<filter>
		<blacklist>
			<directory suffix=".php">system</directory>
			<!--directory suffix=".php">application/libraries</directory-->
		</blacklist>
	</filter>
</phpunit>
```

What this config file is doing:

1. Telling to use a custom bootstrap file
2. Telling PHPUnit to look for tests under application/tests
3. Creating a constant for a PHPUnit runtime test environment, `PHPUNIT_TEST`. This will be used to hack into CI bootstrap behaviour.
4. Creating a global variable $CFG that CI needs available at runtime
4. Providing a `$_SERVER['REMOTE_ADDR']` default value so CI's security checks won't break.
5. For code coverage analysis, we're filtering out CI *system* directory, and optionally your *application/libraries* directory (if you uncomment that line).

### Hacking CI system files ###

CI relies a lot on superglobal variables, which are not easily available in a PHPUnit runtime. However, providing those critical global variables beforehand and providing simple checks using the new `PHPUNIT_TEST` constant adapts CodeIgniter behaviour.

CI will start by doing autoloading, reading config files, and most importantly, **load your default Controller**, since it has no routing information (no URI or CLI parameters). This is actually useful, since this is what makes `$CI =& get_instance()` possible in our tests.

#### Hacks: ####

>

> `CodeIgniter.php`
>> *Line 325 changed to:*
>>
>>```
>> if (!defined('PHPUNIT_TEST')) { ... }
>>```
>>
>> This is the line which starts calling your controller's intended method from URI parameters, but in test we want to call the method ourselves. So this check skips the method call.
>>
>> *Line 386 changed to:*
>>
>>```
>> if ($EXT->_call_hook('display_override') === FALSE && !defined('PHPUNIT_TEST'))
>>```
>>
>> Same logic here, we have to prevent CI from outputting things before our tests.


Tips for Testing
================

### Using provided CITestCase class ###

The `CITestCase` file extends PHPUnit's `PHPUnit_Extensions_Database_TestCase` and provides a few common use cases for CodeIgniter tests, most importantly, database assertions.

The example `EmailHelperTest` provided before would be changed to:

```
<?php

class EmailHelperTest extends CITestCase
{    
	public function setUp()
	{
		$this->CI->load->helper('email');
        $this->CI->load->model('contactmodel');
	}
	
	public function testEmailValidation()
	{
		$this->assertTrue(valid_email('test@test.com'));
		$this->assertFalse(valid_email('test#testcom'));
	}
    
    public function testContactsQty()
    {
        $qty = $this->CI->contactmodel->getContactsQty();
        $this->assertEquals($qty, $this->db->getRowCount('contacts'));
    }
}

?>
```

It provides a property `$this->CI` with the default controller loaded, and another `$this->db` as a wrapper to a `PHPUnit_Extensions_Database_DB_IDatabaseConnection`. If your test does not use database connections, it will not be loaded.

**Considerations about this database connection:** it uses your application's database config file to initiate a PDO "fixture" **from you real database**. In other words, it is not a fixture, but a quick way for you to make assertions in your real database. As it is, you must define a `setUp()` call in your methods, or it will use PHPUnit's default database logic to truncate it after every test.

Make sure you understand [PHPUnit's Database Manual](http://www.phpunit.de/manual/3.7/en/database.html) completely before you use this database connection, and change it to your needs.

### Set $db['default']['db_debug'] to FALSE ###

If you set `$db['default']['db_debug'] = TRUE`, every error your test encounters will output database information and end the script. It is better to throw Exceptions and let you test handle it.

### Avoid die() and exit() ###

If you use them, they'll interrupt testing, as they end PHP execution. That's why `show_error()` and `show_404` were changed to throw Exceptions, which are much easier to test.

### Change your environment to 'testing' ###

In CodeIgniter `index.php`, you can change the application environment from 'development' to 'production' or 'testing', which primarily avoid `error_reporting()` outputs from PHP. If you let `error_reporting()` outputs, you won't be able to use PHPUnit strict mode, and you'll have a hard time testing your own outputs.

### Test show_error() and show_404() using Exceptions ###

You can benefit from the hacks made at bootstrap.php by making this kind of tests:

```
/**
 * @expectedException           PHPUnit_Framework_Exception
 * @expectedExceptionCode       403
 * @expectedExceptionMessage    forbidden
 */
public function testCreateNullName()
{
    // this should call show_error('forbidden', 402)
	$this->CI->resourcemodel->deleteResource(1);
}
```

### Testing your Controllers ###

Testing Controllers are not straightfoward as Models, Libraries or Helpers in CodeIgniter, because CI
has tightly coupled dependencies with classes like Session, Cookies, Caches or Input. It is beyond scope
of this manual to teach how to (re)write CI applications with dependency injection in mind, but if you
have testable controllers, you can load them in tests by changing the `$CI` variable:

```
public function testHomeController()
{
	$CI =& get_instance();
	$CI = new Home();
	$CI->index();
}
```

Changelog
------------

2014-01-21:
* Reduced number of system/core file hacks needed (only 1 now)
* New bootstrap file (hacks into system/core/Common.php)
* Improved phpunit.xml (creates global $CFG so system/core/Utf8.php doesn't need hacking)
* Ensured compatibility with CI 2.1.4, probably compatible with all 2.1.x versions
* Updated README

2013-04-03:
* Initial version
* Compatible with CI 2.1.3