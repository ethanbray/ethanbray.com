---
layout: post
title:  "Unit Testing Zend Framework Forms"
date:   2019-06-15 13:33:10 +0000
categories: [zend, testing]
---

# Unit Testing Zend Framework Forms

The Zend Framework [documentation](https://docs.zendframework.com/tutorials/unit-testing/)
on unit testing focuses on testing controllers rather than 
components such as forms, filters or validators.

Testing these components is vital for catching bugs and 
regressions in the behaviour of key forms in your web application.

It can be unclear how to test these components in the least brittle 
fashion. Elements with special behaviour such as file uploads can be 
particularly tricky to test.

## Testable Behaviour
There are several parts of a form that can be tested.
Tests can be written to ensure the form and its elements
are configured properly, as well as ensuring the filtering and 
validation are behaving correctly.

The construction of a form and a filter can be tested
by simply instantiating the class. You can then test
the individual elements for specific configuration options
such as their type, labels or required state. Any of the options
or attributes can be retrieved by using the `getOption` or
`getAttribute` method on a form element.

```php
use Foo\Bar\ExampleForm;
use PHPUnit\Framework\TestCase;
use Zend\Form\Element\Text;
use Zend\Form\Element\Password;

class ExampleFormTest extends TestCase {
    public function test_username_configuration(): void
    {
        $exampleForm = new ExampleForm();   
        $usernameInput = $exampleForm->get('username');
        
        $this->assertInstanceOf(Text::class, $usernameInput);
        $this->assertEquals('Username: ', $usernameInput->getOption('label'));
        $this->assertTrue($usernameInput->isRequired());
    }
    
    public function test_password_configuration(): void
    {
        $exampleForm = new ExampleForm();   
        $passwordInput = $exampleForm->get('password');
        
        $this->assertInstanceOf(Password::class, $passwordInput);
        $this->assertEquals('Password: ', $passwordInput->getOption('label'));
        $this->assertTrue($passwordInput->isRequired());
    }
}
```

The filtering and validation can be tested by instantiating the input filter, setting
the test data and then retrieving the specific validator you're testing. You can then
assert whether the validation was successful, what the error message(s) are and what the value of the input is post-filtering.

```php
use Foo\Bar\ExampleFormFilter;
use PHPUnit\Framework\TestCase;

class ExampleFormFilterTest extends TestCase {
    public function test_username_validator(): void
    {
        $formFilter = new ExampleFormFilter();
        $formFilter->setData(['username' => 'foobar']);
        $validator = $formFilter->get('username');
        
        $this->assertTrue($validator->isValid());
        $this->assertEquals([], $validator->getMessages());
    }
    
    public function test_username_filter(): void
    {
        // For this test, we're testing that our input is having whitespace trimmed
        // and any tags removed
        
        $formFilter = new ExampleFormFilter();
        $formFilter->setData(['username' => ' foobar <script>']);
        $validator = $formFilter->get('username');
        
        $this->assertTrue($validator->isValid());
        $this->assertEquals('foobar', $validator->getValue());
    }
}
```

I recommend using a [data provider](https://phpunit.readthedocs.io/en/8.1/writing-tests-for-phpunit.html#data-providers)
to try an array of different values. This will help cut down on the amount of duplicated
tests you have to write.

```php
use Foo\Bar\ExampleFormFilter;
use PHPUnit\Framework\TestCase;

class ExampleFormFilterTest extends TestCase {
    /**
     * @dataProvider getUsernameData
     */
    public function test_username_validator($username, bool $expected, array $messages): void
    {
        $formFilter = new ExampleFormFilter();
        $formFilter->setData(['username' => $username]);
        $validator = $formFilter->get('username');
        
        $this->assertEquals($expected, $validator->isValid());
        $this->assertEquals($messages, $validator->getMessages());
    }
    
    private function getUsernameData(): array
    {
        return [
            ['user', true, []],
            ['User', false, ['username' => ['notLowercase' => 'Username must be lower case']]],
            ['1234', true, []],
            ['', false, ['username' => ['isEmpty' => 'Username is required.']]],
            ['^9bhy&', false, ['username' => [
                'notAlnum' => 'Username not valid. Use only alphanumeric characters.',
            ]]],
            ['bl ah', false, ['username' => [
                'notAlnum' => 'Username not valid. Use only alphanumeric characters.',
            ]]],
            [000, false, ['username' => ['stringLengthTooShort' => 'Username not valid. Minimum length 4 characters.']]],
            ['a', false, ['username' => ['stringLengthTooShort' => 'Username not valid. Minimum length 4 characters.']]],
            ['123456789012345678901234567890123456789012345678900', false, ['username' => [
                'stringLengthTooLong' => 'Username not valid. Maximum length 15 characters.']]],
        ];
    }
}
```

## Testing File Elements and Validators
Testing elements such as file uploads can be slightly trickier due to the 
more complex validators such as `Size` or `WordCount`.


You could test these validators by using example files that you
commit to your repository. However, this bloats your
repository with large amounts of files.

We can use [vfsStream](https://github.com/bovigo/vfsStream) as a stream wrapper
for a virtual file system. This allows us to programatically create virtual files
that can then be used during our tests.

```
use Foo\Bar\ExampleFormFilter;
use org\bovigo\vfs\content\LargeFileContent;
use org\bovigo\vfs\vfsStream;
use PHPUnit\Framework\TestCase;

class ExampleFormFilterTest extends TestCase {
    public function test_file_size_validator_fails(): void
    {
        $exampleFile = vfsStream::newFile('example_file.csv')
            ->withContent(LargeFileContent::withMegabytes(2));
    
        $formFilter = new ExampleFormFilter();
        $formFilter->setData(['file' => $exampleFile]);
        $validator = $formFilter->get('file');
        
        $this->assertFalse($validator->isValid());
        $this->assertEquals(
            ['size' => 'File size must be under 1MB'],
            $validator->getMessages()
        );
    }
    
    public function test_file_size_validator_passes(): void
    {
        $exampleFile = vfsStream::newFile('example_file.csv')
            ->withContent(LargeFileContent::withMegabytes(1));
            
        $formFilter = new ExampleFormFilter();
        $formFilter->setData(['file' => $exampleFile]);
        $validator = $formFilter->get('file');
                
        $this->assertTrue($validator->isValid());
        $this->assertEquals([], $validator->getMessages());
    }
    
    public function test_file_extension_must_be_csv(): void
    {
        $exampleFile = vfsStream::newFile('example_file.zip')
            ->withContent(LargeFileContent::withMegabytes(1));
            
        $formFilter = new ExampleFormFilter();
        $formFilter->setData(['file' => $exampleFile]);
        $validator = $formFilter->get('file');
                        
        $this->assertFalse($validator->isValid());
        $this->assertEquals(
            ['fileExtensionFalse' => 'File has an incorrect extension'],
            $validator->getMessages()
        );
    }
    
    public function setUp(): void
    {
        vfsStream::setup();
    }
}
```

The virtual files can be treated exactly like normal files, allowing you to test 
validators such as `WordCount` easily. Documentation for vfsStream can be found
[here](https://github.com/bovigo/vfsStream/wiki).