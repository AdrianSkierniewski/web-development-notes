Testing

This document contains notes related to testing, using these tools:
    TDD (Test Driven Development)
    Mockery (for creating mock objects)
    phpunit
    Selenium
    Laravel techniques:
        call/response
        web crawler
        IoC/facades
        In-memory database and test environment
        Testing with an array repository




Test Driven Development (TDD) 
--------------------------------------------------

This is a development technique that helps create more reliable code over time.
Begin by writing a test that will fail
Test it (make sure it fails)
Write the code to make the test pass
Test it (make sure it passes)
Refactor (make sure all tests still pass)

Mocks
-------
A mock is a replacement for an object that we can use for testing.
Rather than hitting an actual class (eg, to write to the database), go to a mock
Integration testing should hit actual classes

Mockery is a project that makes creation and handling of mocks easier:
in composer.json, require  "mockery/mockery": "dev-master"

Using Mocks:
(instructions from https://tutsplus.com/tutorial/better-testing-in-laravel/)
My controller is ItemsController
My model is Item

I'll set up several different classes:
ItemsController
Item
ItemRepositoryInterface
EloquentItemRepository

In ItemsController, use this:

    public function __construct(ItemRepositoryInterface $items)
    {
        $this->items = $items;
    }

    Interface ItemRepositoryInterface
    {
        public function all();
        public function find($id);
        ...
        any other functions you want the controller to be able to access
    }

    class EloquentItemRepository implements ItemRepositoryInterface
    {
        public function all(){     return Item::all(); }
        public function find($id){ return Item::find($id); }
        ... 
        other functions, just call the corresponding item function
    }

The application also needs to know the default repository to use to implement the interface:

    App::bind('ItemRepositoryInterface', 'EloquentItemRepository');

When testing, using Mockery,

    public function testMockShow()
    {
        $mock = \Mockery::mock('ItemRepositoryInterface');
        $mock->shouldReceive('find')->once()->andReturn('{"name":"works"}');
        App::instance('ItemRepositoryInterface', $mock);

        $response = $this->call('GET', 'items/1');
        $this->assertTrue($response->isOk());
        $this->assertNotEmpty($response->getContent());
        $json = json_decode($response->getContent());
        $this->assertEquals('works', $json->name);
    }

This should bypass the database completely.
In some cases, we need to return an intermediate object (eg, items/1/vendors)

    public function testShowItemVendors()
    {
        $mockVendor = $this->mock('Vendor');
        $mockVendor->shouldReceive('get')->once()->andReturn('{"name":"vendor works"}');

        $mockItem = $this->mock('Item');
        $mockItem->shouldReceive('find')->once()->andReturn($mockItem);
        $mockItem->shouldReceive('vendors')->once()->andReturn($mockVendor); 

        $response = $this->call('GET', 'items/1/vendors');
        $this->assertTrue($response->isOk());
        $this->assertNotEmpty($response->getContent());
        $json = json_decode($response->getContent());
        $this->assertEquals('vendor works', $json->name);
    }

    public function mock($class)
    {
        $repo = $class . 'RepositoryInterface';
        $mock = \Mockery::mock($repo);
        App::instance($repo, $mock);       
        return $mock;
    }



phpunit
----------
I'm using phpunit for testing. It will read configuration information from phpunit.xml, in the root directory of the project.

Incomplete tests:
Skip them by putting this at the beginning of the test:
$this->markTestIncomplete();


We can only test some groups at a time:

    /**
     * @group database
     * @group remoteTasks
     */
    public function testSomething()
    {
    }

testSomething() is now in two groups, and if either is added on the command line (or in the config.xml) --exclude-group parameter. it won't be run.

This won't run integration tests tagged with @group integration:

    phpunit --exclude-group integration         

This will run only tests tagged with @group integration:

    phpunit --group integration                 

Tags for phpunit can be put on either classes or individual functions. This means we can make a tag for the current test, and move it to new tests as we work on them. Then, run phpunit for the current test only, bypassing all other tests. (it's a good idea to run all tests from time to time, like just before I take a short break).

    phpunit --group now

We can also include or exclude specific test groups in the phpunit.xml file:

    <?xml version="1.0" encoding="UTF-8"?>
    <phpunit backupGlobals="false"
            backupStaticAttributes="false"
            bootstrap="bootstrap/autoload.php"
            colors="true"
            convertErrorsToExceptions="true"
            convertNoticesToExceptions="true"
            convertWarningsToExceptions="true"
            processIsolation="false"
            stopOnFailure="false"
            syntaxCheck="false"
    >
        <testsuites>
            <testsuite name="Application Test Suite">
                <directory>./app/tests/</directory>
            </testsuite>
        </testsuites>

        <groups>
          <exclude>
            <group>integration</group>
            <group>failing</group>
          </exclude>
        </groups>

    </phpunit>

(this will automatically exclude the integration and failing test groups, that is anything tagged with @group integration  or  @group failing)

To use a different configuration file, use:

    phpunit -c other_config_file.xml

To generate documentation, use:

    phpunit --testdox

To show code coverage, use:

    phpunit --coverage-html <folder>

You can then to go the specified folder for marked-up coverage. It shows lines of code that have been run in green, other lines in red. It will also show everything in the framework that has been run. 

    
Make a test dependent on success of a previous test:
 
    public function testEmpty()
    {
        $stack = array();
        $this->assertTrue(empty($stack));
        return $stack;   // also sends this variable to any following tests - if this worked
    }

    /**
     * only runs if testEmpty() passed
     *
     * @depends testEmpty
     */
    public function testPush(array $stack)
    {
    }



Selenium
-------------

We can test the actual user experience with selenium. To set up a selenium test, use this:

    class SeleniumTest extends PHPUnit_Extensions_Selenium2TestCase
    {
        public function __construct()
        {
            parent::__construct();
            $this->setBrowserUrl('http://lkata');
            $this->setBrowser('chrome');
            $this->setHost('localhost');
            $this->setPort(4444);
        }

        public function testHomePageDoesNotIncludeDebugError()
        {
            $this->assertEquals(0, 
                preg_match('/xdebug-error/i', $this->source()), 
                'should not return an xdebug error');
        }
    }




Laravel Techniques
==========================

Call/Response
---------------
When unit testing, we can call functions and get the responses. We can also do this with mocks. The basic format is like this:

    $response = $this->action('GET', 'ItemsController@show', array('1'));
    $response = $this->call('GET', '/items/1');
    $this->assertTrue($response->isOk());
    $this->assertNotEmpty($response->getContent());

The response class has several useful functions, including:

    getStatusCode()
    getContent()                    // returns string with final html page 
    getOriginalContent()            // returns View that creates final html page
    headers->get('header-name')

Response Helper Functions:

    Function Name           Status Codes
    isInvalid()             <100 and >=600
    isInformational()       >=100 and <200
    isSuccessful()          >=200 and <300
    isRedirection()         >=300 and <400
    isClientError()         >=400 and <500
    isServerError()         >=500 and <600
    isOk()                  200
    isForbidden()           403
    isNotFound()            404
    isRedirect()            201,301,302,303,307,308
    isEmpty()               201,204,304
    isNotModified()
 
use like:

    if ($response->isOK)  doSomething;

The view class that is returned also has valuable information:

    $result->original->getData();               // data sent to the view
    $result->getOriginalContent()->getData();   // data sent to the view (same)
    $result->original->getEngine();             // templating engine (eg, blade)
    $result->original->getName();               // name of the view (eg todo/index)



Web Crawler
-------------
The Symphony web crawler component will go through the DOM of the page to handle very specific test cases. It can be used during integration tests to simulate a browser (much faster than Selenium).

We can see if there is a div with id 'item' like this:

    public function testSampleItemIsInAnItemDiv()
    {
        $crawler = $this->client->request('GET', '/');
        $this->assertGreaterThan(0, $crawler->filter('div.item')->count());
    }

We can get information for children in the DOM chain, with:

    $children = $crawler->filter('div.item')->first()->children();
    echo $children->filter('h4')->count();
    foreach($children as $child) {
        echo($child->tagName);
        echo($child->getAttribute('class'));
        echo($child->nodeValue);
    }

Other stuff:

    // Click a link:

        $crawler = $this->client->request('GET', '/user/login');
        $link = $crawler->filter('a:contains("Greet")')->eq(1)->link();
        $crawler = $client->click($link);
     
    // Find the submit button on a form:

        $form = $crawler->selectButton('submit')->form();
     
    // set some values

        $form['name'] = 'Lucas';
        $form['form_name[subject]'] = 'Hey there!';
     
    // submit the form

        $crawler = $client->submit($form);
      
    // Assert that the response matches a given CSS selector.

        $this->assertGreaterThan(0, $crawler->filter('h1')->count());
 
Test against the Response content directly if you just want to assert that the content contains some text, or if the Response is not an XML/HTML document:

    $this->assertRegExp(
        '/Hello Fabien/',
        $client->getResponse()->getContent()
    );
 
Force each request to be executed in its own PHP process to avoid any side-effects when working with several clients in the same script:

    $client->insulate();
 
// Search through all content on page for a string

    $crawler = $this->client->request('GET', '/user/login');
    $this->assertTrue($this->client->getResponse()->isOk(), 'should have an OK response');
    $this->assertContains('Please Log In', 
        $this->client->getResponse()->getContent(), 'should contain "Please Log In" ');
 
// Find a specific string in a specific location

    $crawler = $this->client->request('GET', '/user/login');
    $this->assertCount(1, $crawler->filter('h1:contains("Please Log In")'), 
        'should contain "Please Log In" in h1 tag (only once)');
 
Useful Assertions:

    // Assert that there is at least one h2 tag with the class "subtitle"
    $this->assertGreaterThan( 0, $crawler->filter('h2.subtitle')->count());
 
    // Assert that there are exactly 4 h2 tags on the page
    $this->assertCount(4, $crawler->filter('h2'));
 
    // Assert that the "Content-Type" header is "application/json"
    $this->assertTrue(
        $client->getResponse()->headers->contains(
            'Content-Type',
            'application/json'
        )
    );
 
    // Assert that the response content matches a regexp.
    $this->assertRegExp('/foo/', $client->getResponse()->getContent());
 
    // Assert that the response status code is 2xx
    $this->assertTrue($client->getResponse()->isSuccessful());
 
    // Assert that the response status code is 404
    $this->assertTrue($client->getResponse()->isNotFound());
 
    // Assert a specific 200 status code
    $this->assertEquals(200,$client->getResponse()->getStatusCode());
 
    // Assert that the response is a redirect to /demo/contact
    $this->assertTrue($client->getResponse()->isRedirect('/demo/contact'));
 
    // or simply check that the response is a redirect to any URL
    $this->assertTrue($client->getResponse()->isRedirect());
 
Browsing
----------
The Client supports many operations that can be done in a real browser:

    $client->back();
    $client->forward();
    $client->reload();
 
    // Clears all cookies and the history
    $client->restart();
 
Accessing Internal Objects
----------------------------
If you use the client to test your application, you might want to access the client's internal objects:

    $history   = $client->getHistory();
    $cookieJar = $client->getCookieJar();
 
You can also get the objects related to the latest request:

    $request  = $client->getRequest();
    $response = $client->getResponse();
    $crawler  = $client->getCrawler();
 
If your requests are not insulated, you can also access the Container and the Kernel:

    $container = $client->getContainer();
    $kernel    = $client->getKernel();
 
Redirecting
-------------
When a request returns a redirect response, the client does not follow it automatically. You can examine the response and force a redirection afterwards with the followRedirect() method:

    $crawler = $client->followRedirect();
 
If you want the client to automatically follow all redirects, you can force him with the followRedirects() method:

    $client->followRedirects();
 
The Crawler
------------
A Crawler instance is returned each time you make a request with the Client. It allows you to traverse HTML documents, select nodes, find links and forms.
 
Traversing:

Like jQuery, the Crawler has methods to traverse the DOM of an HTML/XML document. For example, the following finds all input[type=submit] elements, selects the last one on the page, and then selects its immediate parent element:

    $newCrawler = $crawler->filter('input[type=submit]')
        ->last()
        ->parents()
        ->first();
 
Many other methods are also available:

    Method                  Description
    filter('h1.title')      Nodes that match the CSS selector
    filterXpath('h1')       Nodes that match the XPath expression
    eq(1)                   Node for the specified index
    first()                 First node
    last()                  Last node
    siblings()              Siblings
    nextAll()               All following siblings
    previousAll()           All preceding siblings
    parents()               Returns the parent nodes
    children()              Returns children nodes
    reduce($lambda)         Nodes for which the callable does not return false
 
Extracting Information:

    // Returns the attribute value for the first node
    $crawler->attr('class');
 
    // Returns the node value for the first node
    $crawler->text();
 
    // Extracts an array of attributes for all nodes (_text returns the node value)
    // returns an array for each element in crawler, each with the value and href
    $info = $crawler->extract(array('_text', 'href'));
 
    // Executes a lambda for each node and return an array of results
    $data = $crawler->each(function ($node, $i)
    {
        return $node->attr('href');
    });

We can chain commands like this (which will assert the last input field in the form has a type of 'submit):

    this->assertEquals('submit', $crawler->filter('form')->filter('input')->last()->attr('type'), 'should have submit button');
 
Links:

To select links, you can use the traversing methods above or the convenient selectLink() shortcut:

    $crawler->selectLink('Click here');
 
    $link = $crawler->selectLink('Click here')->link();
    $client->click($link);
 
Forms:

    $buttonCrawlerNode = $crawler->selectButton('submit'); 



Laravel 4 IoC and Facades
---------------------------
http://www.thenerdary.net/post/30859565484/laravel-4

When using a laravel class, we generally use a facade:

    $var = Session::get('foo');
 
Under the hood, it does this:

    $app->resolve('session')->get('foo');
 
So, we can swap out parts of the framework, like so:

    $app['session'] = function()
    {
        return new MyCustomSessionLayer;
    }
 
For instance, maybe you want to make the whole Redirect layer for a test. In your test you could just do:

    $app['redirect'] = $mock;

For any classes that use facades (most of Laravel's classes), get the original class name with:

    echo get_class(App::getFacadeRoot());       (App, or any other class)


https://news.ycombinator.com/item?id=5044336
However, things like the Input, URL, File, etc. classes still being static could lead to some testability problems. I've broken encapsulation on the Input class just to make it a little more testable. You can set the Input just by saying "Input::$input = array()".

You can swap out entire components with your own. For instance, if you had a class that inherited from the root Response object (Illuminate\Http\Response), you could use it (instead of the standard response class) for all response handling. Just edit app/config/app.php:

    //'Response'        => 'Illuminate\Support\Facades\Response',
    'Response'        => 'Api\Facades\Response',    (your own response facade)

    
    
In-memory database and test environment
-----------------------------------------
An in-memory database is much faster than writing data to your actual database (it doesn't require any disk reads, indexes, etc.)

From: http://net.tutsplus.com/tutorials/php/testing-like-a-boss-in-laravel-models/
Within the app/config/testing directory, create a new file, named database.php, and fill it with the following content:

    // app/config/testing/database.php
    <?php     
    return array( 
        'default' => 'sqlite',
        'connections' => array(
            'sqlite' => array(
                'driver'   => 'sqlite',
                'database' => ':memory:',
                'prefix'   => ''
            ),
        )
    );
 
Before running tests:
Since the in-memory database is always empty when a connection is made, it’s important to migrate the database before every test. To do this, open app/tests/TestCase.php and add the following method to the end of the class:

    /**
     * Migrates the database.
     * This will cause the tests to run quickly.
     *
     */
    private function prepareForTests()
    {
        Artisan::call('migrate');   // sets up all tables
        $this->seed;                // seed test database values
        Mail::pretend(true);        // if using mail
    }

    // When using Mockery, it's important to close it at the end of the test
    public function tearDown()
    {
        \Mockery::close();
    }


Testing with an Arrray Repository
------------------------------------

We can get fast, solid testing with an array repository, which basically stubs the functionality we need. Here's the critical pieces:

    interface TodoRepositoryInterface
    {
        public function all();
    }

    class EloquentTodoRepository implements TodoRepositoryInterface
    {
        public function all()     { return Todo::all(); }
    }

    class ArrayTodoRepository implements TodoRepositoryInterface
    {

        protected $guarded = array();
        public static $rules = array();

        public function all()     
        {
            return array(
                array(
                    'id'            => 999,
                    'title'         => 'foo',
                    'description'   => 'bar',
                ),
            );
        }
    }

    class TodosTableSeeder extends Seeder {

        public function run()
        {
            DB::table('todos')->delete();

            $todos = array(
                array(
                    'id'            => 1,
                    'title'         => 'Learn Laravel',
                    'description'   => 'It\'s really important',
                    'created_at'    => new DateTime,
                    'updated_at'    => new DateTime,
            ...
        }

The global binding is set to this:

    App::bind('TodoRepositoryInterface', 'EloquentTodoRepository');

My controller looks like this:

    class TodoController extends BaseController 
    {

        protected $todo;

        public function __construct(TodoRepositoryInterface $todo)
        {
            $this->todo = $todo;
        }

        public function index() 
        {
            return View::make('todo/index')
                ->with('items', $this->todo->all());
        }
    }

My tester:

    class TodoControllerTest extends TestCase
    {
        public function testTodoControllerExistsAndReturnsView()
        {
            $result = $this->action('GET', 'TodoController@index');
            $this->assertContains('Illuminate\View', get_class($result->getOriginalContent()));
            $this->assertViewHas('items');
        }

        public function testTodoControllerCanLoadDataFromDB()
        {
            $result = $this->action('GET', 'TodoController@index');
            $data = $result->getOriginalContent()->getData();
            $this->assertEquals(1, $data['items'][0]['id']);        
        }

        public function testTodoControllerCanLoadDataFromArray()
        {
            App::bind('TodoRepositoryInterface', 'ArrayTodoRepository');
            $result = $this->action('GET', 'TodoController@index');
            $data = $result->getOriginalContent()->getData();
            $this->assertEquals(999, $data['items'][0]['id']);        
        }
    }

To test specific actions, we can use this:

        $data = $this->action('PUT',                // request type
            'TodoController@update',                // controller/method to use
            array('999'),                           // id sent to update 
            array('title'=>'something new'));       // new data via Input

        $data = $this->action('DELETE', 
            'TodoController@destroy', 
            array(42));

        $result = $this->action('POST', 
            'TodoController@store', 
            array('title'=>'test'));



Additional stuff for testing
------------------------------

To dump a query:

    var_dump(DB::getQueryLog());

This will produce output like this (where you can see the actual query string):

    array (size=1)
      0 => 
        array (size=3)
          'query' => string 'select * from `questions` where `qid` = ? limit 1' (length=49)
          'bindings' => 
            array (size=1)
              0 => int 4
          'time' => string '0.77' (length=4)