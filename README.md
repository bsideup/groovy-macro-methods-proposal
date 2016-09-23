Macro methods proposal for Apache Groovy
---

# TL;DR
This proposal is aimed to integrate https://github.com/bsideup/groovy-macro-methods into the groovy-core.

# Motivation
Groovy has very powerful compile time meta programming system. Right now it represented in two ways - global AST transformations and local AST transformations. 

While local transformations are very useful, they are (obviously) limited to the scope where they are applied. It covers most of the cases, but sometimes the scope is global, so we have to use global AST transformations. Some examples of them:
- Spock
- GORM
- Groovy's Grab
- Griffon
- MacroGroovy (available as `macro` method in groovy-core since 2.5)

During my work on MacroGroovy I discovered the pattern which can be reused - replace method call with AST it produces as a global AST transformation. It's called "Syntactic macros", and it's a common thing for the languages like Lisp, Scala, Rust, Haxe and others. Consider the following:
```java
@Macro // <-- macro method aka magic!
static Expression warn(MacroContext ctx, Expression exp, Expression msg) {
    // compile time property check!
    if ("development" !== System.getProperty("ENV")) {
        return new EmptyExpression()
    }
    
    String src = toCodeSource(ctx);
    return macro {
        !$v{exp} && println($v{src} + ": " + $v{msg})
    }
}
```

And the usage:
```java
int age = 10;

warn(age >= 18, "User is under 18");
```

It will print (in runtime) a message **only if `ENV` system property was equal to "development" during the compilation**. Not the runtime, but compilation time. So, if you compile with "-DENV=development", the code will be transformed to:
```java
int age = 10;

!(age > 18) && println("myFile.groovy:3" + ": " + "User is under 18")
```

But the same code compiled without `ENV=development` it will be equal to:
```java
int age = 10;
```

As you can see, macro methods are being executed at compile time and they must return an AST expression, so compiler will replace such call with it.

Another nice thing that you can get the information about the place where macro code was injected by accessing `MacroContext` (again, at compile time!). In this example we use `toCodeSource(context)` to get the position in source code where warning was called.

---
Good parts so far:
- **Compile-time** meta programming
- Access to the place where macro method was called
- Almost **as simple as method call**, but with all the power of compile-time meta programming

# Usage for library developers
Macro methods are useful if you develop a library as well. For instance, imagine you're writing ORM library, let's call it "GroORM" :)

We want to provide a method to perform compile-time, type safe queries:
```groovy
def targetAudience = User.sqlQuery {
    select(id, name as username) where age > 18 && age < 25 orderBy age
}
```

This can be implemented with Macro method:
```java
@Macro
static Expression warn(MacroContext ctx, ClosureExpression block) {
    def modelClassExp = context.call.objectExpression;
    
    return convertASTToSQLQuery(block);
}
```

And, after compilation, it will look something like this:
```java
def targetAudience = GroORM.executeSQL(User, '''
SELECT
    id, name as username
WHERE
    age > 18 and age < 25
ORDER BY
    age
''')
```
It can also check that User class has `id`, `name`, `password` and `age` fields at compile time. 

Even more - it can check that age is number type, for instance. 

The result's type will be `List` because GroORM can parse SQL expression at compile time and make some assumptions about the result.

---
To sum up:
- Libraries can implement powerful, type-safe, compile-time conversions without manually dealing with Global AST transformations.
- Runtime performance is great because transformations are being done in compile time.
- Thanks to Groovy's flexible syntax, one can implement really powerful expressions like an example above - `name as username` is something common for SQL. We have `as` operator in Groovy, so, until "Semantic analysis" phase, it's just a CastExpression!

# Current status
Currently proposal is implemented as a 3rd party library: https://github.com/bsideup/groovy-macro-methods

This is a global AST transformation and all macro methods share the same transformation (vs "Global transformation per use case")

For each static method marked with `@Macro` annotation it will create an internal Groovy Extension Method with the same name and signature:
```java
(MacroContext, ...argExpressions)
```

So, if we have 
```java
@Macro
static Expression mySuperMethod(MacroContext ctx, ConstantExpression constExp)
```

we will get:
```java
mySuperMethod("Hello") // match
mySuperMethod(123) // match

mySuperMethod(123, "Hello") // no match, too many arguments
mySuperMethod(prefix + " World!") // no match, argument is CallExpression
mySuperMethod {} // no match, argument is ClosureExpression
mySuperMethod() // no match, arguments are empty
```

# Try it yourself
You can try Macro methods right now with the latest Groovy release. They are implemented as a library for now.

For instance, here is an example of Pattern Matching for Groovy implemented with Macro methods:
https://github.com/bsideup/groovy-pattern-match

It depends on `groovy-macro-methods` available in JCenter:
https://github.com/bsideup/groovy-pattern-match/blob/d7c4c5494238b84b7685ce5fc433c8676416f1dc/build.gradle#L24-L24
```gradle
dependencies {
    compile 'ru.trylogic.groovy.macro:groovy-macro-methods:0.2.0'
}
```

It implements macro method `match`:

https://github.com/bsideup/groovy-pattern-match/blob/d7c4c5494238b84b7685ce5fc433c8676416f1dc/src/main/java/ru/trylogic/groovy/pattern/PatternMatchingMacroMethods.java#L38-L41

So you can write Groovy code with compile-time pattern matching:
```java
def fact(num) {
    return match(num) {
        when String then fact(num.toInteger())
        when 0 or 1 then 1
        when 2 then 2
        orElse it * fact(it - 1)
    }
}

assert fact("5") == 120
```

(For more examples check the tests: https://github.com/bsideup/groovy-pattern-match/blob/e2c2b0472b7d078a8b299085705584cfea827405/src/test/groovy/ru/trylogic/groovy/pattern/PatternMatchingMacroMethodsTest.groovy)

# Open questions
- Syntax: https://github.com/bsideup/groovy-macro-methods-proposal/issues/1
- Global nature: https://github.com/bsideup/groovy-macro-methods-proposal/issues/2