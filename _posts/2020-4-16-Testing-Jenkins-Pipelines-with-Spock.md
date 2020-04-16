---
layout: post
title: "Testing Jenkins Pipelines with Spock"
categories: [programming]
tags: [ programming, devops, jenkins, groovy, spock]
---

In todays post I'd like to go through Jenkins pipeline unit testing using jenkins-spock library, it's written from groovy newbie viewpoint, and mostly using examples to show how to do things. I left some parts as direct quotes, and hopefully didn't forget any sources, enjoy.

Post has two parts, [Spock](#spock) with just enough spock explanation to write tests, and [Jenkins-Spock](#jenkins-spock-library) with jenkins-spock library quick tour.

# Spock
Jenkins-spock uses standard groovy spock for test structure:

You need to extend `Specification` class if you are testing groovy classes, or `JenkinsPipelineSpecification` if you are testing JenkinsFiles, and vars scripts.

That is fairly standard, and similar to JUnit, 

Tests themselves live inside feature methods which are named using strings.
And are broken into parts via blocks.

### Simple test:

```
// test suite
class MyFirstTest extends JenkinsPipelineSpecification {
  
  def "Test Name"():     // test definition
    expect:              // block
        1 == 1           // implicit assertion
}
```


### Blocks:

Spock supports BDD style testing via blocks out of the box. 6 blocks are available:
`given, when, then, expect, cleanup, where`

* `given` : setup phase of a test, also everything before any block is implicitly moved in given block. equals `setup:`
* `when` : do something with the system under test
* `then` : test the response - implicitly assertions
* `expect` : do something and test response - if you don't like given, when, then - implicit assertions
* `cleanup` : cleanup phase
* `where` : data driven testing

Extending our example:
```
class MyFirstTest extends JenkinsPipelineSpecification {
  
  def "Test Name"():     // test definition
    given:
      int left = 1 
      int right = 1
    
    when:
      int results = left + right

    expect:
        result == 2           // implicit assertion
}
```

all boolean expressions inside expect and then blocks are asserted, you can use groovys `assert` keyword to check expressions anywhere else.

### Spec setup methods:

To surprise nobody there are methods to run before each feature method, once per spec, and same for cleanup:

```
def setupSpec() {}    // runs once -  before the first feature method
def setup() {}        // runs before every feature method
def cleanup() {}      // runs after every feature method
def cleanupSpec() {}  // runs once -  after the last feature method
```

### Data Driven testing:

If you need to test more conditions at once, use `where:` block,several syntactic ways to use it, my favorite is data table:

```
class MathSpec extends Specification {
  def "maximum of two numbers"(int a, int b, int c) {
    expect:
    Math.max(a, b) == c

    where:
    a | b | c
    1 | 3 | 3
    7 | 4 | 7
    0 | 0 | 0
  }
}
```

more here: [http://spockframework.org/spock/docs/1.3/data_driven_testing.html](http://spockframework.org/spock/docs/1.3/data_driven_testing.html)

### Cardinality:

Spock supports cardinality testing with `<int> *` syntax: 

```
1 * subscriber.receive("hello")      // exactly one call
0 * subscriber.receive("hello")      // zero calls
```
[http://spockframework.org/spock/docs/1.3/interaction_based_testing.html#_cardinality](http://spockframework.org/spock/docs/1.3/interaction_based_testing.html#_cardinality)

### Mocking:
> Spock has it's own mocking framework, making use of interesting concepts brought to the JVM by Groovy. First, let's instantiate a Mock:
>
>`PaymentGateway paymentGateway = Mock()`
>
>In this case, the type of our mock is inferred by the variable type. As Groovy is a dynamic language, we can also provide a type argument, allow us to not have to assign our mock to any particular type:
>
>`def paymentGateway = Mock(PaymentGateway)`
>
>Now, whenever we call a method on our PaymentGateway mock, a default response will be given, without a real instance being invoked:
>
>```
>when:
>    def result = paymentGateway.makePayment(12.99)
> 
>then:
>    result == false
>```
>The term for this is lenient mocking. This means that mock methods which have not been defined will return sensible defaults, as opposed to throwing an exception. This is by design in Spock, in order to make mocks and thus tests less brittle.

[https://www.baeldung.com/groovy-spock#2-mocking-using-spock](https://www.baeldung.com/groovy-spock#2-mocking-using-spock)

### Stubbing:
>We can also configure methods called on our mock to respond in a certain way to different arguments. Let's try getting our PaymentGateway mock to return true when we make a payment of 20:
>
>```
>given:
>    paymentGateway.makePayment(20) >> true
> 
>when:
>    def result = paymentGateway.makePayment(20)
> 
>then:
>    result == true
>```
>What's interesting here, is how Spock makes use of Groovy's operator overloading in order to stub method calls. With Java, we have to call real methods, which arguably means that the resulting code is more verbose and potentially less expressive.
>
>Now, let's try a few more types of stubbing.
>
>If we stopped caring about our method argument and always wanted to return true, we could just use an underscore:
>
>`paymentGateway.makePayment(_) >> true`
>If we wanted to alternate between different responses, we could provide a list, for which each element will be returned in sequence:
>
>`paymentGateway.makePayment(_) >>> [true, true, false, true]`
>There are more possibilities, and these may be covered in a more advanced future article on mocking.

[https://www.baeldung.com/groovy-spock#3-stubbing-method-calls-on-mocks](https://www.baeldung.com/groovy-spock#3-stubbing-method-calls-on-mocks)

## JUnit vs Spock:

Although Spock uses a different terminology, many of its concepts and features are inspired by JUnit. Here is a rough comparison:

|Spock                 |JUnit|
|:---------------------:|:---:|
|Specification         |Test class
|`setup()`             |`@Before`
|`cleanup()`           |`@After`
|`setupSpec()`         |`@BeforeClass`
|`cleanupSpec()`       |`@AfterClass`
|Feature               |Test
|Feature method        |Test method
|Data-driven feature   |Theory
|Condition             |Assertion
|Exception condition   |`@Test(expected=...)`
|Interaction           | Mock expectation (e.g. in Mockito)



# Jenkins-Spock library

Now we know how to write basic spock tests, lets move to jenkins part.

A tiny bit about Jenkins pipelines,
Jenkins pipelines comes in 3 structures:

* classes
* pipeline variables (vars/something.groovy) 
* pipeline scripts (whole Jenkinsfiles)

Classes are standard groovy classes, and are testable without anything special using standard spock unit tests.
The other too are a bit different, and have global variables that are coming from Jenkins, that needs to be mocked.
Those global variables, are essentially of 3 types:
* actual global variables set by Jenkins for every job:
[https://jenkins.io/doc/book/pipeline/jenkinsfile/#using-environment-variables](https://jenkins.io/doc/book/pipeline/jenkinsfile/#using-environment-variables)
* other pipeline vars from shared library you are in, or implicitly loaded shared library
* pipeline steps from plugin or Jenkins

In order to run jenkins pipeline code without Jenkins, we need to mock every global variable so we don't get undefined reference compilation errors.
All the mocking is done by `JenkinsPipelineSpecification` class, and we need to extend it to write tests.

This class ensures that all pipeline extension points exist as Spock Mock objects so that the calls will succeed and that interactions can be inspected, stubbed, and verified. You can access a Spock mock for any pipeline step that would exist by using `getPipelineMock("object")`.


## Mock Pipeline Steps
Mock pipeline steps are available at `getPipelineMock("stepName")`. You can verify interactions with them and stub them:
 
```
then:
	1 * getPipelineMock( "echo" )( "hello world" ) // check that echo was called with hello world once
  //stubs sh call when called with echo hi, to return hi
	1 * getPipelineMock( "sh" )( [returnStdout: true, script: "echo hi"] ) >> "hi"
```

For example, the node(...) { ... } step's body is automatically executed:
```
when:
	node( 'some-label' ) {
		echo( "hello" )
	}
then:
	1 * getPipelineMock("node")("some-label) // test that node was called with 'some-label`
	1 * getPipelineMock("echo")("hello") // test that echo was called with 'hello'
```

## Pipeline Vars:

Jenkins pipeline scripts need a special treatment, because they contain global variables provided by Jenkins Plugin, Jenkins itself, and potentially implicitly loaded shared libraries. Because of that we need to mock them, and stub them.


`loadPipelineScriptForTest()`:

loads our script for testing, and enables us to run them with arguments:

```
def MyFunction = loadPipelineScriptForTest("vars/MyFunction.groovy")
MyFunction('test arg')
```

If we have some Jenkins global env var in the script, we need to set it to something:
```
MyFunction.getBinding().setVariable( "BRANCH_NAME", "master" )
```

Method calls on GlobalVariables are available as mocks at `getPipelineMock("VariableName.methodName")`

### Stubbing pipeline vars:

stubbing is done like for all mocks, just stub .call method:
```
given:
  //when MyFunction gets called with Hello, return Hello World.
	getPipelineMock("MyFunction.call")("Hello") >> "Hello World"
when:
  //run your script with MyFunction call:
	Jenkinsfile.run()
then:
  // echo was called once with Hello World
	1 * getPipelineMock("echo")("Hello World")
```

## Pipeline Scripts:

You can also test whole pipeline JenkinsFiles, only difference is you have to call `.run()` on them, after loaded from `loadPipelineScriptForTest()`

```
def "Jenkinsfile"() {
	setup:
		def Jenkinsfile = loadPipelineScriptForTest("com/homeaway/CoolJenkinsfile.groovy")
	when:
		Jenkinsfile.run()
	then:
		1 * getPipelineMock("node")("legacy", _)
		1 * getPipelineMock("echo")("hello world")
}
```


## Explicit mocks:

If for some reason you won't get automatic mock for a variable, for example from a plugin you don't have in a dependencies, you can explicitly mock them by: `explicitlyMockPipelineStep("varName")`


This is about it for unit testing jenkins pipelines with jenkins-spock, I personally quite enjoy the implicit mocks for almost everything, and BDD variant of Spock. All of it seems a lot more lightweight then using JenkinsPipelineUnit.


#### Sources:

1. [http://spockframework.org/spock/docs/1.3/spock_primer.html](http://spockframework.org/spock/docs/1.3/spock_primer.html)
2. [https://javadoc.io/static/com.homeaway.devtools.jenkins/jenkins-spock/2.1.2/com/homeaway/devtools/jenkins/testing/JenkinsPipelineSpecification.htm](https://javadoc.io/static/com.homeaway.devtools.jenkins/jenkins-spock/2.1.2/com/homeaway/devtools/jenkins/testing/JenkinsPipelineSpecification.htm)
3. [https://www.baeldung.com/groovy-spock](https://www.baeldung.com/groovy-spock)
4. [https://github.com/ExpediaGroup/jenkins-spock](https://github.com/ExpediaGroup/jenkins-spock)