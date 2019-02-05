---
layout: post
title:  "Associative arrays: associated with abuse"
date:   2019-02-04 22:15:00 -0800
categories: programming
---
_Associative arrays_, often called _dictionaries_ or sometimes _hashes_ after the popular implementation of them, are useful little critters. They pair a unique thing of some type, usually a string, the _key_, with another thing of a possibly different type, the _value_. And called dictionaries by way of analogy to looking things up by a known name. Another analogy might be with a phonebook, because I might ask some particular associative array about a Billy Pigslinger and it would respond with Billy Pigslinger's phone number.

In strongly-typed languages, my phonebook array might be constrained. It might only accept a name presented in a particular way as the key and it might only reply with valid phone numbers as values. It's not always worth the effort but programmers sometimes invent these contracts to prevent accidents and make guarantees; to prevent a confused user from indexing into the phonebook with a hamburger and expecting to get a duck back. The contract says that the software will complain if you try to plug in a hamburger and the guarantee says you will always get a phone number or nothing back, never a duck.

PHP is not generally strongly typed (though the language has some facilities to help you out; more on that later) and that's a virtue in some ways but it lays the groundwork for some lazy code. An example I came across recently in production code (though names and locations have been changed) begins as a typical PHPism:

{% highlight php %}
static function createSomeObject ($array = []) {
{% endhighlight %}

Where `$array` is some variable collection of named parameters. This pattern is so common in PHP code that it's downright idiomatic. The default value is a lie and this function will never succeed with an empty array, as you will now see.

{% highlight php %}
  if (empty($array['type'])) {
    throw new SomeException("Parameter 'foo' is required.");
  }

  $validFields = {'type', 'foo', 'bar', 'baz', 'qux', 'extra'};

  $obj = new SomeObject;

  foreach ($validFields as $field) {
    if (isset($array[$field])) {
      $obj->$field = $array[$field];
    }
  }
{% endhighlight %}

We're starting to raise some fundamental questions. Field `foo` is required but the function signature does nothing to communicate or enforce this; should it? If only some fields are valid and there's a tidy object that encapsulates all of them, why not use the object in the first place and skip the array ceremony?

{% highlight php %}
  if (!$obj->type instanceof Type || !$obj->bar instanceof Bar /* || others */) {
    throw new InvalidArgumentException("Parameter was supposed to be something but it isn't.");
  }
{% endhighlight %}

The function's opacity at this point is maddening. The type validating is coming in hard but late and no linter can save us.

Note that `$obj` is an instance of `SomeObject` and can be initialized with invalid member properties long before an exception is raised. That's a recipe for spooky action at a distance. The type validation should be moved into `SomeObject`'s setters and that ridiculous loop should be eliminated so we're not blindly copying data over. Give `SomeObject` a chance to defend itself!

Finally and catastrophically, at some point, maybe multiple points, it's going to do something like this,

{% highlight php %}
  // $array['extra'] has some parameters specific to the type,
  // dramatized here with a simple function and unnecessary parameters
  // for illustrative purposes.
  evaluateExtraParameters($obj->type, $obj->extra);
{% endhighlight %}

I say _at some point_ because in this case, the action was deferred in time and space: not present in the function but happening somewhere else, sometime else.

This function is a jungle of mysteries. It leaves the user with a lot of detective work to do.

1. What are valid arguments for this function?
2. Which arguments are required for this function to finish?
3. What are valid arguments in `extra` and which of those are required for this particular instance of `Type` to work as it should?
4. What type are any of these parameters supposed to be and how do I instantiate them?
5. What is the practical purpose or justification for any of the above?

Some of these questions can (and should) be answered in the documentation, but this is PHP we're talking about. The documentation is just thinner up here. Even if it were, there's value in putting the requirements of a function front-and-center, part of the broader idea of _self-documenting code_. That's not in lieu of documentation, it's an addition to it, in an ideal world.

Arrays lend themselves well to certain use cases.

* The data is of the same type and is having some repetitive operation performed on it, also known as _map and reduce_.
* The data is stringy and something repetitive is done with those strings.
* The data is named string properties of a consistent pattern and you do something repetitive with that, like an array of named links each of which has properties like `href`, `visited`, `tooltip`, and so on.
* Universally, the contents of the array should be _optional_. The absence of something should not make the operation die.

Essentially, I strongly feel that a function should only die on absent data if it warned me in its contract that it would. Flouting this makes it a _bad function_.

The example function above is flagrant array abuse and should be refactored to take an instance of a class. Specifically, it should be an abstract class or an interface and all of the different `type`s can subclass or implement it. Or it can make a mess with enumerations and callback functions, I do not care so long as it doesn't make me guess.

And as luck would have it, typed arguments have been available in PHP since version 5. Violations of this have resulted in a fatal error since PHP 7. It's easier than ever to communicate requirements.

For the sake of completion,

{% highlight php %}
  return $obj;
}
{% endhighlight %}

I don't know what I'm supposed to do with my new `SomeObject`. It belongs in a ~~museum~~ _package-internal array_, funny enough.