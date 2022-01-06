# "Call" Operator (`fn::()` == `fn.call()`) for Javascript

* Stage: 0
* Champions: @tabatkins, @js-choi, others?

---

* [Motivation](#motivation)
* [Proposal](#proposal)
* [Comparison With Other Proposals](#comparison-with-other-proposals)
	* [bind-this operator](#bind-this)
	* [Extension operator](#extensions)
	* [Partial Function Application](#partial-function-application)

Motivation
==========

There are good reasons to want to "extract" a method from a class, and then later call that method, either on instances of the extracted class, or on unrelated objects that mimic the original class enough to be useful context objects.

For example, `Object.prototype.toString()` is a useful method for doing cheap, easy "brand"-like identification (not *reliable*, but useful), but won't be callable on any object whose class provides its own `toString()` method, or which has a null prototype.

For another example, most of the Array methods are intentionally defined to be generic and usable on any "array-like" with a `.length` property and numeric properties storing data. Reusing them, rather than writing your own equivalents by hand, is a reasonable thing to do.

Finally, some "defensive" coding styles want to guard against prototype mutation in general, and thus want to pull significant methods off of objects early in a script's lifetime, and then use them later rather than relying on the prototype still having the correct methods on it.

Currently, all of these use-cases can be handled by calling the method with `.call()`, like:

```js
const {slice, map} = Array.prototype;
const skipFirst = slice.call(arrayLike, 1);
```

This usage is in fact *very common*, particularly among "utility" code, as shown [by studies of large bodies of extant JS code](https://github.com/tc39/proposal-bind-this/issues/12). It's so common that it's probably worthwhile to make easier and more reliable.

Proposal
--------

We add a new calling-syntax operator, `fn::(receiver, ...args)`, which invokes function with the given receiver and arguments, identical to `fn.call(receiver, ...args)`.

Precedence is the same as normal function calling; the `::(` is a singular token treated identically to `(` when doing a normal function call. This is the same as the optional-call operator `.?()`, or partial function application `~()`.

Example usage:

```js
const {slice, map} = Array.prototype;
const skipFirst = slice::(arrayLike, 1);
const mapped = map::(arrayLike, x=>x+1);

const {toString} = Object.prototype;
const brand = toString::(randomObj);
```

Comparison With Other Proposals
-------------------------------

### Bind-This

The call operator is very similar to (and occupies roughly the same syntax space as) [the bind-this `::` operator](https://github.com/tc39/proposal-bind-this).
Rather than being based on `.call()`, bind-this is based on `.bind()`; `receiver::fn` is identical to `fn.bind(receiver)`. You can immediately invoke the bound function, as `receiver::fn(args)`, giving the same behavior as this proposal, but can do somewhat more as well.

So on the surface, the bind-this operator appears to completely subsume the call operator. However, I believe the call operator is the better choice, for a few reasons.

1. Parsing/precedence is simpler. The bind-this operator operator has to put restrictions on the RHS of the operator in order to have reasonable and predictable parsing: only dotted ident sequences are allowed, with parens required to do generic operations. For example, if you need to fetch the method out of a map, you can't write `newReceiver::fnmap.get("theMethod")`, because it's unclear whether the author means `fnmap.get("theMethod").bind(newReciever)` or `fnmap.get.bind(newReciever)("theMethod")` (aka a `.call()`). In fact, this sort of expression is still absolutely *valid*, it'll just always be interpreted as the second possibility, possibly being a confusing runtime error.

	On the other hand, `fnmap.get("theMethod")::(newReciever)` is immediately clear and distinct from `fnmap.get::(newReceiver, "theMethod")`, with no chance of confusion. You know precisely what expression is getting invoked to provide the method, and what arguments are being passed to it.
	
2. Bind-this overlaps with [partial function application](https://github.com/tc39/proposal-partial-application). PFA makes it extremely easy to hard-bind a method to the object it's currently on - just write `obj.meth~(...)` and you're done, since the reciever is automatically captured by PFA when creating the temporary function. Bind-this can do this too, just more verbosely: `obj::obj.meth`, which is annoying to write and potentially problematic if `obj` is actually a non-trivial expression that now needs to be written and executed twice.

3. Bind-this overlaps with [the pipe operator `|>`](https://github.com/tc39/proposal-pipeline-operator). Using bind-this, one can write and invoke "methods" without having to add them to an object's prototype. For example:

	```js
	function zip(that, fn) {
		return this.map((e, i)=>{
			return fn(e, that[i]);
		});
	}
	[1,2,3]::zip([4,5,6], (a,b)=>a+b);
	
	// vs, with pipe
	function zip(arr1, arr2, fn) {
		return arr1.map((e, i)=>{
			return fn(e, arr2[i]);
		});
	}
	[1,2,3] |> zip(##, [4,5,6], (a,b)=>a+b);
	```
	
	Note that the `zip()` function had to be written to specifically expect and use its `this` binding, 
	making it inconvenient and awkward to invoke in any way *other than* with this operator;
	you *must* call it either as `a::zip(b, fn)` or, verbosely, as `zip.call(a, b, fn)`.
	
	On the other hand, the pipe version is just a normal function, which can be called with pipe if you want a "method-chaining"-esque experience,
	or just as `zip(a, b, fn)` if that's not necessary and calling it normally is fine and clear.
	
	This was already one of the concerns cited when discussing pipe operator semantics,
	as a strike against the F#-style pipe.
	Authors would be incentivized to write libraries *intended* for the F#-style pipe,
	which are less convenient to invoke any other way
	(`a |> f(b)` with pipe, or `f(b)(a)` without);
	this potential ecosystem forking was seen as a problem to avoid.
	The adoption of Hack-style pipe syntax avoided this entirely,
	but bind-this brings it back.
	
	Note that this proposal does *not* introduce such problems;
	while a library author *could* still write free functions that rely on a `this` binding being provided,
	calling it would be done as `zip::(a, b, fn)`,
	which is identical but slightly less convenient to just writing the function as taking all its arguments normally
	so it's invokeable as `zip(a, b, fn)`.
	Thus there's simply no reason for an author to write their library that way,
	and the ecosystem-forking concerns are minimal.
	
### Extensions

The [Extensions operator](https://github.com/tc39/proposal-extensions) is essentially just bind-this's "methods without having to mutate prototypes" functionality
(aka the "bind, but immediately invoke" syntax),
with some more functionality.
It suffers from identical overlap and issues as bind-this.
	
### Partial Function Application

No direct overlap - PFA creates new functions by binding some of the arguments of an existing function, possibly including the receiver of a method.
It doesn't directly invoke functions.

The only conceptual overlap is that PFA makes it easy to hard-bind a method to the object it's already on,
like `const fn = obj.meth~(...); fn(a, b, c);`,
and this proposal allows one to extract a method from a class
and then call it on objects of that class,
but that's only similar at a high level;
in practice the two use-cases are quite distinct.

On the plus side, the two proposals potentially work together quite well--
PFA does *not* allow one to hard-bind a method against *a different object*,
like `.bind()` allows,
but it can easily be defined to work together with the call operator to achieve this:
`meth::~(newReciever, ...)` is now equivalent to `meth.bind(newReceiver)`.
