Roy
===

Notes from Functional Brighton joint meetup with Async JS, 28 June 2012.

Richard Dallaway, @d6y


What is it?
-----------


[A small functional language that compiles to JavaScript](http://brianmckenna.org/
).

* It's a functional language
* Statically typed
* Has type inference
* Aims to have readbale JavaScript output


Plan: very brief show what it looks like, say why it's interesting, give specific examples of the interesting bits.

Most of this is from the IEEE paper paper.  Check the _Links_ at the end.


What does it look like?
-----------------------

Interact via the REPL, `roy -r` to run code, or compile and see the JS output.

	$ ./roy
	Roy: Small functional language that compiles to JavaScript
	Brian McKenna <brian@brianmckenna.org> (http://brianmckenna.org/)
	:? for help
	roy> let double x = x * 2
	roy> double 21
	42 : Number

Interacting with JavaScript:

	$ cat example.roy
	let double x = x * 2
	
	console.log (double 21)
	
	$ ./roy -r example.roy
    42

or...
	
	$ ./roy example.roy
	Roy info: Identifier "console" on line 2 is unknown - dynamically typed
	
	$ cat example.js
	var double = function(x) {
	    return x * 2;
	};
	console.log(double(21));
	//@ sourceMappingURL=example.js.map
	
	
	$ node example.js
	42


There are a billion things in JavaScript that Roy won't know about. `console` is an example: it is treated as untyped and just output as is.


What's interesting about it?
----------------------------

* It's not trying to port an existing language and libs (that'd be a lot of work)

* It's not ignoring JavaScript.  Anything not recognised is assumed to be a JavaScript entity.

* Written in JavaScript, so you can compile in the browser.

* Types - you write type annotations when you want to restrict; no more falsyness.

* Structural types

* Option

* Pattern matching

* Possibilities for making async call code readable.


Using it
========


Types & typechecking
--------------------

First, saves you from things like:

	js> 7 + true + "hi"
	"8hi"

	roy> 7 + true
	Error: Type error on line 0: Boolean is not Number


Type inference:

	roy> 7
	7 : Number

This is compiled to a Javascript `var`:
	
	roy> let greeting = "Hello"
	roy> :t greeting
	String
	
Fucnctions:
	
	roy> let double x = x * 2
	roy> :t double
	Function(Number, Number)
	
	roy> let identity x = x
	roy> :t identity
	Function(#ba, #ba)
	
	roy> let repeat s n  = if n == 0 then s else s ++ (repeat s n - 1)
	roy> repeat "*" 10
	*********** : String
	roy> :t repeat
	Function(String, Number, String)
	
	roy> repeat 10 "*"
	Error: Type error on line 0: Number is not String


Adding types:

Boolean, Number, String, Structure, Array, Tuple, Function

	roy> type Person = {name: String, age: Number}
	
	roy> let birthday (p: Person)  = p.age + 1
	roy> :t birthday
	Function({name: String, age: Number}, Number)
	
	roy> birthday {name:"Bob", age:99}
	100 : Number
	
	roy> birthday 7
	Error: Type error on line 0: Number is not Person

Structure type are useful with type inference too, if you leave off the `p: Person` in `birthday`:

	roy> birthday {name: "Bob", age: 99}
	100 : Number
	
	roy> birthday {fruit: "Banana", age: 99}
	100 : Number


Duck typing. Anything that has an age property will work here, which is a little safer than passing arbitrary objects around and hoping they have the properties you want. Good for JSON?

Can do `{name: "Bob"} with {age: 99}`


Lists & pattern matching
-------------------------

I've drawn a [Cons diagram](http://tinyurl.com/celala6) if you need it.

Cons data type, map, filter...

	data List a = Nil | Cons a (List a)
	
	let nil = Nil()
	
	let xs = Cons 1 (Cons 2 (Cons 3 nil))
	
	console.log(xs)
	
	let map f l = match l
	  case (Cons v r) = Cons (f v) (map f r)
	  case Nil = nil
	
	let filter f xs = match xs
	  case Nil = nil
	  case (Cons h t) = if (f h) then 
	        Cons h (filter f t)
	      else 
	        filter f t
	
	let head xs = match xs
	 case (Cons h t) = h
	
	let double x = x * 2
	
	let rs = filter (\x -> x > 3) (map double xs)
	
	let rs = filter (\x -> x > 3) (map (\x -> x * 2) xs)
	
	console.log rs
	
	let pretty xs = match xs 
	  case Nil = ""
	  case (Cons h t) = h ++ " " ++ (print t) 
	
	console.log (pretty rs)



Option & the M word
-------------------

	data Option a = Some a | None
	
	let optionMonad = {
	  return: λx →
	    Some x
	  bind: λx f → match x
	    case (Some a) = f a
	    case None = None ()
	}
	
	let m = (do optionMonad
	  x ← Some 1
	  y ← Some 3
	  return x + y
	)
	
	match m
	  case (Some x) = console.log x
	  case None = console.log "I need both x and y"
	
	
	let m2 = (do optionMonad
	  x ← Some 1
	  y ← None ()
	  return x + y
	)
	
	match m2
	  case (Some x) = console.log x
	  case None = console.log "I need both x and y"

Produces...

	$ node test/fixtures/good/option_monad.js
	4
	I need both x and y


Anything with `bind` and `return` function is Roy's monad representation.

Note the Unicode.

JQuery
------

Chain async calls:

	let print x = console.log(x)
	
	let deferred = {
	  return: $.when
	  bind: \x f ->
	    let defer = $.Deferred ()
	    x.done (\v -> (f v).done defer.resolve)
	    defer.promise ()
	}
	
	let v = do deferred 
	  ip <- $.ajax 'http://cfaj.freeshell.org/ipaddr.cgi'
	  country <- $.ajax 'http://api.hostip.info/country.php'
	  return (ip ++ country)
	
	v.done print

Compiles to:

	var print = function(x) {
	    return console.log((x));
	};
	var deferred = {
	    "return": $.when,
	    "bind": function(x, f) {
	        var defer = $.Deferred();
	        x.done(function(v) {
	            return f(v).done(defer.resolve);
	        });
	        return defer.promise();
	    }
	};
	var v = (function(){
	    var __monad__ = deferred;
	    
	    return __monad__.bind($.ajax('http://cfaj.freeshell.org/ipaddr.cgi'), function(ip) {
	        
	        return __monad__.bind($.ajax('http://api.hostip.info/country.php'), function(country) {
	            
	            return __monad__.return((ip + country));
	        });
	    });
	})();
	v.done(print);
	//@ sourceMappingURL=deferredmonad.js.map



Status
=======

Initial commit was May 2011.  Brian McKenna.  Experimental.


Links
=====

* [IEEE article](http://brianmckenna.org/blog/roy_ieee) is a great starting point for the language. Highly recommended.
* Seriously, read that IEEE article.
* [The blog](http://brianmckenna.org/blog) has lots of short feature introduction posts.
* [Roy home page](http://roy.brianmckenna.org/) contains an in-browser kind of REPL.
* [Latest source](https://bitbucket.org/puffnfresh/roy) is on BitBucket.
* ...but confusingly [also on GitHub](https://github.com/pufuwozu/roy) -- the issues on GitHub contain lots of discussion and ideas.
* There's a [bundle for Sublime and Textmate](https://github.com/paulmillr/roy.tmbundle).
* The [mailing list](groups.google.com/group/roylang) is on Google Groups.
* [Video interview](http://blogs.atlassian.com/2011/11/introduction-to-a-new-javascript-language-roy-with-brian-mckenna/) with Brian McKenna Nov 2011. 
* [Another video](http://www.youtube.com/watch?v=Grgz5yBhvRo) 
* @puffnfresh and @roylangjs on Twitter