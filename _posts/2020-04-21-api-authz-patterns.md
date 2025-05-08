---
layout: post
title:  "Code Patterns for API Authorization: Designing for Security"
date:   2020-04-21 09:00:00 -0500
categories: authz authorization appsec
---

This post was originally at <https://research.nccgroup.com/2020/04/21/code-patterns-for-api-authorization-designing-for-security/> but has been reuploaded due to linkrot.

## Summary

This post describes some of the most common design patterns for authorization checking in web application code. Comparisons are made between the design patterns to help understand when each pattern makes sense as well as the drawbacks of the pattern.

For developers and architects, this post helps you to understand what the different code patterns look like and how to choose between them. For security auditors, the most effective approaches to auditing authorization controls are explained based on which pattern the code uses.


## Code Patterns for Safe APIs

In the course of around 5 years at NCC Group, I’d estimate that I’ve worked on more than 50 source-code-assisted web application assessments. One major takeaway I’ve had is that the security of large applications is a reflection of the coding patterns used in the internal codebase. If there are dangerous (code) APIs available internally, security issues will show up externally when those APIs are used without safety checks.

This isn’t a new idea – awareness of unsafe code APIs has been around for many years. Probably one of the earliest examples was the industry push to migrate C code to using safer functions when operating on strings and arrays (e.g. strcpy vs. strcpy_s). More recently, React chose the name [`dangerouslySetInnerHTML`](https://reactjs.org/docs/dom-elements.html#dangerouslysetinnerhtml), instead of `innerHTML`, in order to better warn developers aware of the security issues with that API.

The same idea applies to the code used in a web application when implementing authorization controls. When designing a new application, it’s important to create APIs which enforce safe data access across the codebase. The APIs determine whether vulnerabilities are introduced that allow attackers to bypass the application’s authorization controls, escalating privileges or attacking other users. Everyone makes mistakes, but safe APIs make it easier to code without causing vulnerabilities, making the application much more secure.


## Common Internal Design Patterns for Authorization

Among the many applications I’ve reviewed, there are generally four code patterns that most applications use to implement authorization. Each code pattern can work in the right situation, but picking the wrong code pattern generally results either in vulnerabilities or prevents the authorization logic from working well for the application’s use cases.


### Ad-hoc

In an ad-hoc pattern, each route defines its own permissions checks and logic.

~~~rb
@route("/messages/:id")
def get_message(context, message_id):
 message = get_message(message_id)

 if context.user == message.recipient:
   return message

 return "Not authorized"
~~~

This is probably one of the most common and most dangerous patterns. It might work in a very simple application, but as the application’s size increases it quickly becomes unmanageable. It’s way too easy (even for a security-conscious developer) to make a mistake that results in a serious authorization bypass, especially when working on a route with years of built-up logic. Consider this the “no guardrails” pattern of authorization logic.

When auditing large applications using this pattern, it is generally very difficult to perform a systematic review of authorization via source code. Instead, manual testing from a user perspective is generally easier. Tools like [AutoRepeater](https://github.com/nccgroup/AutoRepeater) or [Autorize](https://portswigger.net/bappstore/f9bbac8c4acf4aefa4d7dc92a991af2f) can help to accelerate testing. Once an issue or unexpected behavior occurs, it becomes more productive to review the route logic to identify the cause and impact.

## Route-based

With the route-based pattern, each route explicitly declares what permissions are required to access the route. The declaration generally occurs in a decorator or in a routing file, and middleware checks the authorization before proceeding to the route’s logic.

~~~rb
@route("/messages/:id")
@authorize("read_messages")
def get_message(message_id):
  return get_message(message_id)
~~~

This pattern can work in some scenarios, but many applications will run into serious problems. First, there are almost always use cases where you need more context to inform the decision on authorization. If you want to check whether a user has access to a specific object, then you have to implement it in the route logic and create the same ad-hoc pattern as above. This contextual logic could be implemented in middleware, but that often just moves the problem elsewhere.

A secondary, fixable problem with this pattern is the risk of a “fail-open” design. When implementing the decorators and middleware, the tendency is to only perform a check if a route has a specific authorization control applied. However, it’s very easy for a developer to forget to add the decorator to a route, resulting in an authorization vulnerability. To fix this, implement an anti-decorator (like @dangerous_noauth) and make sure that the middleware fails-closed by rejecting authorization to any route without a decorator applied.

Route-based authorization checks are generally most effective in enterprise applications which have two features:

* Strong separation between tenants of the application (i.e. separate installations or databases)
* Objects belong to the tenant, rather than to specific users

Those two features mean that routes generally don’t need to check logic based on a specific user context. Instead, they only need to check that a user has the permission needed to perform some specific action. For example, an Admin has all permissions, but an Accountant can only read financial data.

This pattern is generally straightforward to audit as long as the pattern is searchable. Grep the codebase for all route definitions with a few lines of context and then simply check each definition for the correct permission check. Make sure that the code search doesn’t miss route definitions where no authorization check is applied because of the “fail-open” issue explained above.

### Centralized

The centralized pattern implements authorization in a single location that defines permissions on objects based on roles and context. The rules might be defined in a configuration file or in code-based logic. All route logic calls the centralized APIs when it wants to access (create, read, update, delete) an object. As a result, there’s no way to bypass the centralized authorization logic by accident.

The following example is loosely inspired by Rails and the [CanCanCan](https://github.com/CanCanCommunity/cancancan) gem.

~~~rb
# External-facing route for /message/:id
def get_message(context, message_id)
 # Any logic here is safe as long as we use the safe APIs
 return safe_read_message(context.user, message_id)
end

# Safe APIs replace direct ORM calls
# These methods are usually not handwritten, but this shows the logic
def safe_read_message(user, message_id)

 # Unsafe direct ORM call, not used outside of this method
 message = Message.unsafe_find(message_id)

 # .can? explicitly checks authorization via authorize
 if user.can?(:read, message)
   return message

 raise NotAuthorizedException
end

# Central Authorization Logic
def authorize(user)
 # Examples: users can read profiles, update their own profile, write messages
 can :read, UserProfile
 can :update, UserProfile, id: user.profile.id
 can :create, Message

 # Users can only read messages where they are the recipient
 readable_messages = Message.where(recipient: user)
 can :read, Message, id: readable_messages
end
~~~

This pattern is generally much more effective than an ad-hoc or route-based pattern, in my experience. The advantage of a centralized pattern is that it is extremely easy to audit: all authorization logic is in a single place, and route logic can be forbidden from unsafely accessing objects. With this pattern, developers can code with the knowledge that their own logic won’t bypass any authorization checks as long as they use safe APIs. Checks for unsafe access can also be implemented as part of code review or in continuous integration pipelines.

The downside to this pattern is that it becomes difficult to scale as the number of objects and developers increases. The logic of the single authorization method generally grows linearly with the number of objects. Complex checks further inflate the amount of logic in the authorization method. For these reasons, most codebases generally migrate from a centralized to an object-based model once the centralized method becomes too large to manage.

This is probably the easiest pattern to audit because all authorization checks exist in a single location. However, the codebase should also be searched for usage of unsafe APIs, which act as an escape hatch to the centralized logic. If an unsafe API is used in a route, the route should receive additional focus to ensure bypasses aren’t present.

### Object-based

The object-based pattern is fundamentally similar to the centralized pattern, but permissions are defined on each object (i.e. one authorization method or config file per object). This is more scalable as the number of objects increases.

class UserProfile

 def authorize(user)
   can :read, UserProfile
   can :update, UserProfile, id: user.profile.id
 end
end

class Message

 def authorize(user)
   can :create, Message

   readable_messages = Message.where(recipient: user)
   can :read, Message, id: readable_messages
 end
end

The primary downside to this pattern is that it multiplies the number of places where an authorization issue could occur by the number of object types. With this pattern, it becomes much more important to implement SDLC gates around the authorization logic:

    Extra code review or scrutiny when creating or modifying authorization methods
    Custom static analysis tools which identify unsafe authorization methods
    Authorization-aware test frameworks which enable developers to easily test authorization

When auditing this pattern, typically the same approach is used as with the centralized pattern. However, object-based logic is usually present in much larger and more complex codebases. As a result, I’ve tended to focus on specific features and only read the authorization logic for the objects associated with each feature in isolation. Trying to systematically audit the authorization logic for every object at once is nearly impossible because there is too much code to keep in your head and the relationships between objects are not clear without actually using the application.

## Conclusion: How to Design Safe Authorization APIs

So, when designing your authorization model, which pattern should you choose? Of course, it depends on the type of application, use cases, and functionality, and how the application is expected to scale in terms of number of objects, routes, and developers. More complex applications are usually better off with the centralized or object-based patterns. For simpler applications, route-based or ad-hoc patterns may be acceptable – more complex authorization systems mean more development and maintenance work.

Beyond knowing the specific patterns, I have a few pieces of advice which can apply to any application:

* Identify the use cases of the application and what authorization checks need to be implemented to support those use cases
* Start with a simple design which matches those use cases, but ensure it can scale if you expect the application to grow
* Make sure your design works everywhere and is easy to use, so developers aren’t tempted to bypass the system’s safety checks (at least not without review)
* Try to keep the logic auditable so that you can easily identify places where authorization checks are missed or bypassed
