# call/cc: It's that callback thing you want in Javascript

Even if you're used to passing around functions as values, there are some related concepts that can be very elusive. One of them is ``call/cc'', which is a call that ``bundles the current continuation, and passes it as a function''. It exists in functional programming languages like Scheme, but the idea is more generally useful. But exactly what is is, and why should you care? I'll try to argue that the idea is actually something you're quite familiar with, if you've done a bit of coding with callbacks in Javascript.

Suppose you've got some Javascript code that gets a value (e.g. a date) and does something with it (e.g. takes the first three characters and displays them the user):

    function displayDay() {
      var numChars = 3; // Some computation
      var d = new Date().toString(); // Get a value
      var day = d.substr(0, numChars); // Some more computation
      alert(day + " is the day of " + d);
      return day; // Keep going wherever this function was called.
    }
    displayDay();

(You can open your browser console and try it.)

Now, suppose that you need to get that information from somewhere else on the internet, like <http://www.garron.us/api/cors/date/>. There are a few way to implement this. They're generally unwieldy, but they share one thing in common: they use a *callback* to handle the response.

    // A function that gets a URL via AJAX, and calls the callback with the response.
    function cURL(callback, url) {
      var xhr = new XMLHttpRequest();
      xhr.open("GET", url, true);
      xhr.onload = function() {
        if (xhr.readyState === 4)
        callback(xhr.response);
      }
      xhr.send();
    }
    // Let's try it:
    cURL(alert, "http://www.garron.us/api/cors/date/");

Suppose we wanted to rewrite `displayDay` using such a `cURL` call. In order to use the result from the call, we'd have to take the ``rest'' of the function and turn it into a callback:

    function callback(numChars, d) {
      var day = d.substr(0, 3);
      alert(day + " is the day of " + d);
      // We can't actually return from displayDayWithCallback() here.
    }
    function displayDayWithCallback() {
      var numChars = 3;
      cURL(callback.bind(this, numChars), "http://www.garron.us/api/cors/date/");
      // We don't want to return yet.
    }
    displayDayWithCallback();

This is also called a continuation, because `callback` represents the place where the computation is continued after the AJAX call returns. In this case, the continuation is just what the program would have done *inside* displayDay; it is the continuation of displayDay() at the *current* point when we try to get the data using AJAX. Ideally, we'd really just like to write the following:

    // magicCC doesn't actually exist in Javascript.
    function displayDayCC() {
      var numChars = 3;
      var d = magicCC(cURL("http://www.garron.us/api/cors/date/"));
      var day = d.substr(0, numChars);
      alert(day + " is the day of " + d);
      return day; // Keep going wherever this function was called.
    }
    displayDayCC();

  However, we can't do this, because an AJAX call *has* to take a callback. This is where call/cc - "call with current continuation" comes in. In the code above, `magicCC` should be a function that takes the current continuation of the computation and wraps it up in a function that is passed to somewhere else, which *may* (but doesn't have to) call it back:

    function displayDayCCExpanded() {
      var numChars = 3;
      function currentContinuation(result) {
        var d = result;
        var day = d.substr(0, numChars);
        alert(day + " is the day of " + d);
        return_original(day); // This would actually return where displayDayCCExpanded() was called.
      }
      cURL(currentContinuation, "http://www.garron.us/api/cors/date/");
      // Don't actually return yet.
    }
    displayDayCCExpanded();

This is what I consider to be the essence of ``call with current continuation''. It's that thing you want in Javascript when you're making asynchronous (e.g. AJAX) calls but basically just want to use them synchronously. Javascript forces you to use some sort of callback, when you really just want an implicit callback based on the rest of your function: the current continuation.

I've glossed over a few syntax details because Javascript doesn't actually support a mechanism for call/cc, and so any metaphor is technically inaccurate. In particular, it doesn't emphasize that the continuation is more like a proper tail call and [*doesn't return*](http://www.youtube.com/watch?v=3VMSGrY-IlU#t=34s). It also doesn't explain what should happen if we actually want the original function call to return, and how this would interact with other language features.

Call/cc is really a mechanism that starts a new execution stack, but saves the current one with a function (in a closure) so that calling that function is *just like returning*. But it takes the idea of a this computation and makes it abstract: a computation in a continuation becomes a thing that can be manipulated. A continuation might never be called, or it could be called *multiple times*. Multiple computations could be handled like threads, etc. Some very interesting ideas come out of this, and [Wikipedia](http://en.wikipedia.org/wiki/Callcc) does a decent job of covering them.