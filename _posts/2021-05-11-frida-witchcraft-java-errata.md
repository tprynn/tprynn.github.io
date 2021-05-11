---
layout: post
title:  "The School of Frida Witchcraft: Java Spellcasting Errata"
date:   2021-05-11 18:00:00 -0500
categories: frida android
---


Frida is a dynamic instrumentation framework: it allows you to carve into an
application's internals, mess around with objects, change function logic,
and more. If you haven't used it before, check out:

* The [Official Frida Documentation](https://www.frida.re/docs/home/)
* 11x256's [Frida Tutorial](https://11x256.github.io/Frida-hooking-android-part-1/)
* 11x256's [Frida Android Examples](https://github.com/11x256/frida-android-examples)
* [Awesome Frida](https://github.com/dweinstein/awesome-frida), a list of Frida projects and tools
* [Frida CodeShare](https://codeshare.frida.re/)

Having used Frida extensively to hook into Android applications, there are a
number of lessons I've learned about using Frida productively. Consider this post
to be a list of Frida *errata* - tips and tricks that the documentation doesn't
cover that will make your life easier.

## Lessons Learned

### Keep Things Simple By Sticking to JavaScript

Frida offers a number of language bindings. These language bindings are code
that runs on your own computer to control Frida. But for most applications,
it's not necessary to use a binding and is easier to just use the Frida CLI
to inject a script into a running application. In fact, many examples
that show how to use Frida use a language binding for no reason (including the
official Frida Android [example](https://www.frida.re/docs/examples/android/)).

Unless you need to hook an application immediately at start up, you can usually
just start the application manually and then use the CLI to inject a script:

~~~bash
# connect to Android device via USB (-U) and load (-l) a script
# ensure com.example.myapp is running in the foreground first
frida -U com.example.myapp -l myscript.js
~~~

### Use try/catch Liberally

You should almost always wrap any intercepted function in a try/catch block.
If an exception is raised that isn't caught by you, you could crash the application
or Frida, causing you to waste time getting back to a working state.

~~~js
var MyClass = Java.use('com.example.MyClass')
MyClass['myFunction'].implementation = function() {
  try {
    // your logic
  } catch(err) {
    console.log(String(err.stack))
  }
}
~~~

### Log Hard, But Make Sure to Log Right

When intercepting function calls, there's no debugging functionality built-in to
Frida. This can make life difficult, especially when working with objects and
types that you don't have much insight for. I use two methods to inspect unknown
objects. This one is for Java objects and prints the fields and methods of the
object:

~~~js
function inspectObject(obj) {
  Class = Java.use("java.lang.Class")
  const obj_class = Java.cast(obj.getClass(), Class)

  const fields = obj_class.getDeclaredFields()
  const methods = obj_class.getMethods()
  console.log("inspecting " + tmp_obj.getClass().toString())
  
  console.log("\tfields:")
  for (var i in fields) {
    console.log("\t\t" + fields[i].toString())
  }
  console.log("\tmethods:")
  for (var i in methods) {
    console.log("\t\t" + methods[i].toString())
  }
}
~~~

To see the internals of a Frida Java object, the following method can be used:

~~~js
function inspectJsObject(obj) {
  console.log(JSON.stringify(obj))
  for (var prop in obj) {
    // ignore internal properties, touching them can be dangerous
    const val = prop.startsWith("$") ? "" : obj[prop]

    if (obj.hasOwnProperty(prop)) {
      console.log("\tself." + prop + " : " + val)
    } else {
      console.log("\t" + prop + " : " + val)
    }
  }
}
~~~

Finally, be careful about what you log, especially with strings or byte arrays.
One of the more annoying crashes I experienced occurred when I passed
non-printable characters to `console.log()` - I'd recommend converting any data
that possibly contains binary data to hex or base64 before logging it.

### Safely Handle Native Objects

There are many situations where you have to deal with or create a native (Java)
object. Native objects should be thought of like loose sticks of dynamite:
always at risk of exploding unless properly handled. Whenever you hold a
native object (most commonly, by intercepting a function), perform an explicit
cast to the expected type. Otherwise, things can quickly go sideways due to
polymorphism.

For example, let's consider a message queue:

~~~js
// maybeMessageQueue is the intercepted object holding Messages
const Queue = Java.use("java.util.Queue")
const messageQueue = Java.cast(maybeMessageQueue, Queue)

// Grab the top message without removing it from the queue
const maybeMessage = messageQueue.peek()
const Message = Java.use("com.someapplication.messaging.Message")
const message = Java.cast(maybeMessage, Message)

// Without the explicit cast, we might call the wrong toString or just crash
console.log("Next message: " + message.toString())
~~~

This style of coding feels ponderous, but trying to chain function calls like
`messageQueue.peek().toString()` to avoid casting always gives me more pain than
effort saved.

### Watch for Side Effects

Many functions have side effects that can break further logic flow when called.
In the previous example, if we call `messageQueue.poll()` (instead of `messageQueue.peek()`),
the next message will be removed from the queue and the application will never
end up processing that message. So it's important to be aware of the side effects
of any function calls.

Often, those side effects aren't obvious at first glance. For example, pretend we
are intercepting the `parseResponse` method of `retrofit2.OkHttpCall` (a common
HTTP library):

~~~js
// Call the original implementation of the parseResponse method
const res = this.parseResponse.apply(this, arguments)

// Explicit cast to Response
const Response = Java.use("retrofit2.Response")
const response = Java.cast(res, Response)

// Try to get the response body for logging - side effect empties the body!
// (we should cast here, but it's not important for the example)
const maybeBody = response.body()
console.log(maybeBody.string())

// return the original object
return res
~~~

After the response is returned, logic is returned to the original application.
The application will end up calling `Response.body()` a second time, but our
call to this function actually drained the buffer holding the response body, so
the application will see an empty body or run into an error!

Apart from reverse engineering code, reading documentation, or just making assumptions
based on the method name, the best simple metric I have found to decide whether a
method is likely to be side-effect free is the complexity and statefulness of the
class that defines the method. In my experience, classes which manage lots of internal
state or are passed around to interface with different code components are much more
likely to have unexpected internal behavior.

## Conclusion

Although Frida can be a bit intimidating to start with, it's a powerful tool
that's easily worth the pain. One of the most useful learning resources for me
was looking at other people's explanations and examples, so I hope these lessons
are as helpful to you as those tutorials were to me.
