---
layout: post
title: Enforcing architectural decisions in your tests
date: 2023-06-18
---

## What does the title mean?

When I say 'Enforcing architectural decisions in your tests' what I essentially mean as architects/developers you write explicit automated tests for your code that would run alongside your other test suites (unit, integration etc). These tests can check for things like:

- Dependency references
- Naming of classes/functions
- Classes implement a given interface
- Language constructs (final, sealed)

And basically anything else you can infer from reflection & depending on which library you use.

## Why might you want to do this

When working on a project with multiple developers it's quite normal to write down a set of rules related to the codebase in some form of documentation (naming conventions, where things should go etc).

This is quite important, although not always done, as it makes onboarding people easier. It reduces the amount of time for PR reviews discussing things that are known by more experienced developers in the codebase but possibly not documented. 

I'm a big advocate of automating whatever can be automated in your continous integration pipeline, you wouldn't make reviewers have to manually run the unit tests would you?!

By adding automated test its reduces the chance of human error in PR reviews, we know it's easy to miss things. Additionally this helps new people get up to speed in the codebase by getting nice big test failures when they violate the rules.

As a bonus if you write descriptive tests there is no reason you can't use those test descriptions as automatically generated documentation for your rules.

## How can you do this

It makes sense to define your rules before you start your project. So, lets rely on the TDD process. Write our rules/tests first then start building the project.

We don't want to reinvent the wheel so here are a few libraries for different languages that do almost the same thing:

- C# [https://github.com/BenMorris/NetArchTest](https://github.com/BenMorris/NetArchTest)
- PHP [https://github.com/phparkitect/arkitect](https://github.com/phparkitect/arkitect)
- Java [https://www.archunit.org/motivation](https://www.archunit.org/motivation)

The C# library integrates better with your unit tests and the PHP library is more like a static code analyser and requires a seperate command to run.

These repositories provide good documentation to get you started.

However I'm also going to run through a real scenario using these tools in C# & PHP.

## A basic example

This example won't go into every step you need to take, but it will give you an idea of how rules are defined and what the output looks like when running tests.

If you want more detail and examples you should refer to the documentation of the relevant libraries.

I've also created a example PHP & C# repository on Github:

- [PHP Example](https://github.com/caddoo/php-enforce-architectural-decisions-example)
- [C# Example](https://github.com/caddoo/csharp-enforce-architectural-decisions-example)

### Background

We are creating a new simple API that is responsible for making a chicken 🐓 lay an egg 🥚 and storing a record of the new egg.

The implementation will be incomplete, the purpose here will be to show how to test the architecture.

### Architectural decisions

- We will use a layered architecture with three layers - Infrastructure, Application and Domain.
- Domain can't depend on any other layer.
- Application can't depend on the domain layer.
- Controllers only exist in infrastructure layer.
- Controllers must extend the base controller.

### Defining our rules

{% tabs definingrules %}

{% tab definingrules php %}
```php
<?php
declare(strict_types=1);

use Arkitect\ClassSet;
use Arkitect\CLI\Config;
use Arkitect\Expression\ForClasses\Extend;
use Arkitect\Expression\ForClasses\HaveNameMatching;
use Arkitect\Expression\ForClasses\NotDependsOnTheseNamespaces;
use Arkitect\Expression\ForClasses\NotHaveDependencyOutsideNamespace;
use Arkitect\Expression\ForClasses\ResideInOneOfTheseNamespaces;
use Arkitect\Rules\Rule;

return static function (Config $config): void {
    $mvcClassSet = ClassSet::fromDir(__DIR__.'/../src');

    $rules = [];

    // Dependency rules
    $rules[] = Rule::allClasses()
        ->that(new ResideInOneOfTheseNamespaces('Domain'))
        ->should(new NotHaveDependencyOutsideNamespace('Domain'))
        ->because("The domain layer can't depend on anything");

    $rules[] = Rule::allClasses()
        ->that(new ResideInOneOfTheseNamespaces('Application'))
        ->should(new NotDependsOnTheseNamespaces('Infrastructure'))
        ->because("The application layer cannot depend on the infrastructure layer");

    // Classes in correct layers
    $rules[] = Rule::allClasses()
        ->that(new HaveNameMatching("*Handler*"))
        ->should(new ResideInOneOfTheseNamespaces('Application'))
        ->because("Our command handler should live in application layer");
    $rules[] = Rule::allClasses()
        ->that(new HaveNameMatching("*Command*"))
        ->should(new ResideInOneOfTheseNamespaces('Application'))
        ->because("Our commands should live in application layer");
    $rules[] = Rule::allClasses()
        ->that(new HaveNameMatching("*Controller*"))
        ->should(new ResideInOneOfTheseNamespaces('Infrastructure'))
        ->because("Our controller should live in infrastructure layer");

    // Inheritance rules
    $rules[] = Rule::allClasses()
        ->that(new HaveNameMatching("*Controller*"))
        ->should(new Extend("egg"))
        ->because("egg");

    $config
        ->add($mvcClassSet, ...$rules);
};
```
{% endtab %}

{% tab definingrules C# %}
```csharp

using System.Reflection;
using Application;
using Domain;
using Infrastructure;
using NetArchTest.Rules;

namespace ArchitectureTests;

public class ArchitectureTests
{
    [Fact]
    public void DomainLayerCantDependOnAnyOtherLayer()
    {
        var result = Types.InAssembly(typeof(Chicken).Assembly)
            .Should()
            .NotHaveDependencyOn("Application")
            .And().NotHaveDependencyOn("Infrastructure")
            .GetResult();
        
        Assert.True(result.IsSuccessful);
    }
    
    [Fact]
    public void ApplicationLayerCantDependOInfrastructureLayer()
    {
        var result = Types.InAssembly(typeof(EggHandler).Assembly)
            .ShouldNot()
            .HaveDependencyOn("Infrastructure")
            .GetResult();
        
        Assert.True(result.IsSuccessful);
    }
    
    [Fact]
    public void HandlersShouldOnlyResideInApplicationLayer()
    {
        var result = Types.InAssemblies(_getAssembliesUnderTest())
            .That().HaveNameEndingWith("Handler")
            .Should().ResideInNamespaceMatching("Application")
            .GetResult();
        
        Assert.True(result.IsSuccessful);
    }
    
    [Fact]
    public void CommandsShouldOnlyResideInApplicationLayer()
    {
        var result = Types.InAssemblies(_getAssembliesUnderTest())
            .That().HaveNameEndingWith("Commands")
            .Should().ResideInNamespaceMatching("Application")
            .GetResult();
        
        Assert.True(result.IsSuccessful);
    }
    
    [Fact]
    public void ControllersShouldOnlyResideInInfrastructureLayer()
    {
        var result = Types.InAssemblies(_getAssembliesUnderTest())
            .That().DoNotResideInNamespaceStartingWith("Jetbrains")
            .And().HaveNameEndingWith("Controller")
            .Should().ResideInNamespaceMatching("Infrastructure")
            .GetResult();
        
        Assert.True(result.IsSuccessful);
    }
    
    [Fact]
    public void ControllersShouldInheritBaseController()
    {
        var result = Types.InAssemblies(_getAssembliesUnderTest())
            .That().HaveNameEndingWith("Controller")
            .And().DoNotHaveNameStartingWith("Base")
            .Should().Inherit(typeof(BaseController))
            .GetResult();
        
        Assert.True(result.IsSuccessful);
    }

    private IEnumerable<Assembly> _getAssembliesUnderTest()
    {
        return new[]
        {
            typeof(Chicken).Assembly,
            typeof(EggHandler).Assembly,
            typeof(BaseController).Assembly
        };
    }
}
```
{% endtab %}

{% endtabs %}

### Running now without implementations

Right now our checks pass as there is no code under test.

{% tabs runningwithout %}

{% tab runningwithout php %}

![Running PHPArkitect with no code](/assets/images/running-phparkitect-with-no-code.png)
{% endtab %}

{% tab runningwithout C# %}

![Running netarchtest with no code](/assets/images/running-netarchtest-with-no-code.png)
{% endtab %}

{% endtabs %}

### Writing the code

You can see the code implementation here for PHP and C#:

- [PHP Example](https://github.com/caddoo/php-enforce-architectural-decisions-example)
- [C# Example](https://github.com/caddoo/csharp-enforce-architectural-decisions-example)

### Breaking stuff

Let's violate one of the rules, lets add a controller to our domain layer.

- [Commit in PHP](https://github.com/caddoo/php-enforce-architectural-decisions-example/commit/3aca57cd5aa966c2adf15b1df50e225ed7c74c5f)
- [Commit in C#](https://github.com/caddoo/csharp-enforce-architectural-decisions-example/commit/8cd2f466750ce7dcf3524fcafbb7a233700a50b8)

And run again

{% tabs runningwith %}

{% tab runningwith php %}
![Running PHPArkitect with violations](/assets/images/running-phparkitect-with-violations.png)
{% endtab %}

{% tab runningwith C# %}
![Running NetArchTest with violations](/assets/images/running-netarchtest-with-violations.png)
{% endtab %}

{% endtabs %}

## Rounding up

Making these checks part of your CI will help prevent your lovely original architectural decisions from going off the rails. It's up to you how many rules you add. 

You can write nearly an unlimited amount of rules, so be pragmatic and set rules that are important for you and your project.
