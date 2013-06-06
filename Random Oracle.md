# Random Oracle == memoized rand()

In cryptography, the idea of a [random oracle](http://en.wikipedia.org/wiki/Random_oracle) is very important: it is a function whose output is completely unpredictable, but always consistent.

Of course, any *particular* function is completely predictable, but a "random function" (that we don't know ahead of time) is not.
Usually, we assume we have a black box with such a function, and we can ask for, say, the value at 1.

The black box is an oracle which has an random function in mind, although we don't know anything about that function.
We can go to it and query *"Oh, exalted random oracle, what is the value of this esteemed function at `1`?"* and the oracle will divine the answer for us.
However, this only tells us the value of the function at 1.
If we then ask it the value at 2, we'll be equally clueless about it until we get the answer back.
But if we ask it the value at 1 again, we'll get the same answer as the first time, *since the random oracle is a function* (so it maps every input to a single output, which doesn't change).

## AES: Everyone's Favorite Oracle Family

In practice, we can treat a block cipher like [AES](http://en.wikipedia.org/wiki/Advanced_Encryption_Standard) like a random oracle.
By choosing a random key `K`, we [select a random function](http://en.wikipedia.org/wiki/Currying) `f(x) = AES(K, x)`, which turns any given 128-bit value `x` into a 128-bit output.
We can think of this as "encryption" in the case of AES, but sometimes we only really care about the fact that the output is random-looking.
(This results in some counter-intuitive ideas at the core of secure block cipher modes, but that's not important for this post.)

Well, what is "random-looking"? It means that the outputs appear completely unrelated.
Knowing a bunch of values like `f(1)`, `f(2)`, ..., `f(100)` should give not allow us to guess `f(101)`.
In fact, it shouldn't allow us to know any more information about `f(101)` (e.g.
which values are more likely) than we knew at the beginning.
Of course, as mentioned before, no *single* function has this property.
But if we have a "family" of functions that has this property if we select a function from it at random, this gives us a "PRF": a [Pseudo-Random Function Family](http://en.wikipedia.org/wiki/Pseudorandom_function).

In particular, AES was *designed* to be a PRF (actually a [PRP](http://en.wikipedia.org/wiki/Pseudorandom_permutation), but the distinction is not relevant here), and [no one has made significant progress in breaking it](http://en.wikipedia.org/wiki/Advanced_Encryption_Standard#Security).
If someone can find *any* pattern in the output to AES that makes it a bad choice as a PRF, then it would be a very big deal.
But until someone does, it's good enough for most purposes to assume that AES a PRF and a random oracle.

## Make It Up As You Go Along

One thing that I find very interesting about a (theoretical) random oracle is that its answers can be made up on the fly! Whenever we ask a question like "Oh, exalted oracle, what is the value of `f` at 1?" we have no idea about the output until we hear the answer.
It makes no difference to us whether the oracle already had an idea in mind for `f(1)`, or whether it decided "on the spot" when we asked.

Suppose we keep track of the oracle's answers.
Whenever, we need to know `f(x)` for some value `x`, we do one of two things:
If we haven't asked about `x` before, we consult the oracle, and also write the answer.
However, if we've already asked about `x` before, we just look up the value of `f(x)` we wrote down the first time, and avoid asking the oracle.
This is called *[memoization](http://en.wikipedia.org/wiki/Memoization)*.
It's clear that our memoized lookup process always gives the same answer as the oracle itself.

Now, the only thing we're really doing with the oracle now is asking it for a random value, once per (unique) input.
But if we know the possible output values of the random oracle, we can just do this ourselves! We can emulate an oracle by starting with an empty mapping, and generating + recording an output every time we're asked for an input.
This allows us to build up a (truly) random function as we go along, one that is indistiguishable from an ideal random oracle.
We have turned our ignorance about its output into the ability to emulate it!

## Code Examples

Now, here's the reason that motivated me to start writing this blog post in the first place: Mathematica has a really nice way to memoize the value of a function.
Creating and using a random oracle is as simple as this:

    R[x_] := (R[x] = Random[]);
    
    {R[1], R[2], R[1]}
    (* Example output: {0.226118, 0.563533, 0.226118} *)
    

If you call `R[1]`, Mathematica will generate a random value and return it to you.
But it will also create a special definition for `R[1]` that remembers this value, and use it for `R[1]` any time in the future.
(This is called a "downvalue" for `R`, which you can think of as creating an implicit hashmap entry.
This is actually quite efficient in Mathematica.
You can also view all the memoized values for `R` using `DownValues[R]` and reset them using `Clear[R]`.) If you trust the random number generator to be indistinguishable from random, then this is a perfectly good random oracle that maps anything (Mathematica doesn't even care if it's a number!) to a random number between 0 and 1.

We can do the same in other languages.
For example, we can use a Javascript object to memoize random values.
You can try the following code [in your browser's Javascript console](http://webmasters.stackexchange.com/questions/8525/how-to-open-the-javascript-console-in-different-browsers):

    randomOracle = (function() {
      var r = function(x) {
        if (typeof r.memory[x] === "undefined") {
          r.memory[x] = Math.random();
        }
        return r.memory[x];
      };
      r.memory = {};
      return r;
    })();
    
    console.log([1, 2, 1].map(randomOracle));
    // Example output: [0.3192490176297724, 0.8585976017639041, 0.3192490176297724]
    
    console.log(randomOracle.memory);
    // Example memoized values: Object {1: 0.3192490176297724, 2: 0.8585976017639041}
    

It's also fairly easy in Python:

    import random
    memory = {}
    
    def R(x):
      if not memory.has_key(x):
        memory[x] = random.random()
      return memory[x]
    
    print R(1), R(2), R(1)
    # Example output: 0.528075099578 0.281973619033 0.528075099578
    
    print memory
    # Example memoized values: {1: 0.5280750995781356, 2: 0.28197361903305485}
    

## To Hash or Not To Hash

A pure random oracle provides a nice example of an alternative to hashing in some applications.
Sometimes you don't need a deterministic value for a given input, as long as the same input will always give the same value while your program is running.
For example, I first realized the Mathematica trick while I was implementing tests for a [streaming sketch algorithm](http://en.wikipedia.org/wiki/Count-Min_sketch).
Such algorithms work by taking hashes of values in a stream, and estimating properties by keeping track of them (for example, the smallest hash is `1/(# unique elements)` on average, if your hashes can be interpreted as random values in `[0, 1]`).
I was originally going to pack values into a hash function like SHA256, but it turned out this was completely unnecessary.
The random oracle model can be cleaner and faster to implement when speed and clarity are more important than storage space (e.g. for testing), and can simplify thinking about the important properties of the hash.

You could also *view* a random oracle as a hash function, but this is backwards by crypto conventions: a random oracle is an ideal standard with strong guarantees, and a hash function is a particular instance that tries to live up to some of those standards. (Perhaps a random oracle is a black box with complete mystery inside, while a hash function is a black box with a particular fancy mechanism inside that we try not to care about).

## So?

This is not an especially novel way to view random oracles, but I think it's nice for thinking about crypto when we use a black box as random oracle.
The usual theory-view is that we have a random function plucked from the land of mathematical functions, with the special feature that we get random-looking output at each value.
However, we can create such a "function" by taking over the tasks of selecting the random values, and maintaining the [same-input-same-output property](http://en.wikipedia.org/wiki/Referential_transparency_%28computer_science%29).
This allows us to bring together the coder's notion of a "function" with the mathematical one, just by enforcing the special feature that the output is consistent.
Thus, whether you're reading a theoretical paper or using crypto primitives in practice, the ideal model of a random oracle is really just a memoized function that generates random values.