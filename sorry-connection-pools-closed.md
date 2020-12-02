# Sorry, Connection Pool's Closed

*In a hurry? There's a [TL;DR](#tl-dr) at the bottom.*

In theory, the process of debugging is simple: Isolate the cause of the issue, then correct the mistake in code. Now, most developers will tell you that each of these steps can get quite tricky. Maybe the environment does not allow you to use a debugger to step through code, or you cannot see where the error occurs because it's somewhere deep in a framework you use, or the bug is volatile and only happens under high workload, and so on.

A common shortcut to not have to spend too much time on isolating the cause is using whatever error output you get, e.g. an error code or an exception's message, and searching for that on the web. This is almost scarily effective, as long as the technologies you use are common enough. Someone else will have had your problem before. Someone else will have raised an issue in the issue tracker of the affected software. Naturally, such an effective shortcut is used often, and isolating the cause "the hard way" becomes the exception rather than the norm.

However, relying on what you find during your web search has its risks. I'm not even talking about the case where nobody has had your problem before. Sometimes, you find plenty of results that kind of look like your problem, with suggested and accepted solutions that kind of look like they would work. You try them, but then they don't solve your problem. It's frustrating. Why did it work for these people but not for me? Is my setup different? Am I doing something else wrong? What am I missing?
This is a story about exactly such a scenario, which reinforced a lesson I've already learned from previous bugs:

> The longer it takes to find the cause of a bug, the more obvious and stupid it is.

## The Problem

For my job, I was working on an ASP.NET service, a CRUD API for buildings. Recently, we also added the capability for a user to mark buildings as their favorites, and now I was implementing a feature so that you could get the geolocation (latitude / longitude) of buildings as well. I just finished writing all the tests, my merge request was approved, I deployed it into our testing environment, everything was going smoothly. Then I received a message from the developers that were implementing the frontend that consumed my service:

> Hey, whenever we try to load the frontend, some of the requests fail. Something about "An operation is already in progress".

Alright, looks like my tests didn't cover everything they should have. Reproducing the bug was easy - I started frontend and backend locally, opened my browser, and it performed three requests to my API:

- A request for all the user's favorite buildings
- A request for all the building images
- A request for the geolocation of the buildings

The request for the building data itself goes to a different service.

As reported, some of the requests failed. Different ones each time I loaded the page. Sometimes one, sometimes two. Never all three. A look at the backend console also confirmed their error message: `System.InvalidOperationException: An operation is already in progress`, the stack trace pointing to our database connection being the issue.

Later I got word that this issue already happened *sometimes* before I implemented the geocoding, right around the time when we added the favorites feature. Maybe different requests shared the same database connection?

## Data Access

The building service uses Entity Framework Core (or EF Core). Our pattern for database access was simple. In `Startup.cs`, we registered a `DbContext` for all our repositories to use:

```C# Code
var npgsqlConnection = new NpgsqlConnection(connectionString);
services.AddDbContext<MyDbContext>(options => options.UseNpgsql(
    npgsqlConnection,
    options => options.MigrationsAssembly("Project.Namespace")));
```

In each repository, we can then simply declare a `DbContext` instance, and ASP.NET will inject it for us (whether the repository pattern is actually necessary with `DbContext` is a fair question which will not be addressed here).

## Trying the Online Fix

The first thing I did after confirming the bug report was googling the error message. I found a decent amount of results, and they all boiled down to the same root cause and fix: 

Each `DbContext` instance opens its own connection to the database. Don't use the same instance in multiple threads at the same time. Don't reuse instances after they have been disposed. Always materialize your queries (e.g. call `ToListAsync()`), because otherwise you return an open stream to the database from your method which might be closed somewhere down the line.

I found it a little odd that these were the suggested fixes, because it did not look to me like we were doing any of that (first red flag). On the other hand, I was not sure how exactly the lifetimes of instances injected by ASP.NET behaved (second red flag), so I decided to follow their advice. A lot of this was done while being on call with other developers, so this kept multiple people busy:

- I checked all the repository methods and confirmed we were always materializing the queries.
- I tried using connection pooling by calling `AddDbContextPool()` in `Startup.cs` instead of `AddDbContext()`.
- I tried changing the connection string to the database and explicitly set `Pooling=true` or `Pooling=false`.
- I disabled a database request in the controller's constructor which I thought to maybe interfere with the controller's logic.

Some of these attempts gave me a different error message, but the end result was the same: Some of the requests failed because the database connection was busy. I even tried creating `DbContext` instances manually in each repository method. How can I reuse instances if I create them for every database call? Still, same error. At this point, I was convinced the problem was not in our business logic, but in our ASP.NET configuration.

## Dependency Lifetimes

The building service follows a simple pattern. A request's entry point to the system is the controller, which calls a service to perform the business logic, and the service calls a repository to access the database. As stated before, each repository gets its own `DbContext` instance. All of these services and repositories are injected by ASP.NET.

In ASP.NET, each incoming request gets its own thread, and a fresh instance of the controller that handles the request. When you register a dependency, you have two choices when it comes to the lifetime (I'm discounting the `Singleton` lifetime here since it does not apply): `Scoped` or `Transient`.

A dependency with a `Scoped` lifetime gets created once per request. Let's say you create 10 `Foo` objects in a request, each of which needs a `Bar` object. One `Bar` object will be created and used by all the `Foo` objects.

A dependency with a `Transient` lifetime is created whenever and object that requires this dependency is created. So, if you construct 10 `Foo` objects, 10 `Bar` objects will be created.

All dependencies at that point were registered transient, only the `DbContext` instances were `Scoped` by default. Aha! That must be the crucial difference that's responsible for the error! I specified `ServiceLifetime.Transient` in the `AddDbContext()` method, but nothing changed. Same error.

Out of desperation, I tried every combination of dependency lifetimes I could think of - Setting everything to `Scoped`, setting the services to `Scoped` and the repositories to `Transient`, and so on. Same error.

After running out of combinations, I drew some terrible diagrams of how I thought the dependency lifetimes worked. At this point, I wasted most of my day on this, and a couple hours of other developers' time, who were just as clueless as I was. One controller, one service, one or two repositories. How could there be any overlap? We are not multithreading anything, we are using `await` with all our `async` calls, and we are not manually passing `DbContext` instances anywhere.

## Finding a Solution

At this point, I did what I should have done hours ago. I challenged my assumptions. The only way we could be reusing `DbContext` instances is if different requests used the same controller instance, and thus the same service, repository, and `DbContext` instances. Instead of trying to apply fixes I found on the web, I went on to actually try and understand the problem that was happening.

I started by going into every repository and generating a unique ID in every instance. While doing that, I discovered that each `DbContext` comes with its very own `Guid` out of the box. I added a logging statement to every single repository method that contained the name of the method, as well as the `Guid` of the `DbContext` instance it was using.

Refreshing the frontend, I couldn't believe my eyes: Every repository method logged a different `Guid`! The only exception being two methods that were called from the same repository instance, and I could confirm that these calls were guaranteed to happen sequentially. No `DbContext` instances were shared all along, but all the instances are still apparently sharing one database connection. The dependency lifetimes didn't matter, just as I suspected after making those diagrams.

Dreading having to dive through the Postgresql manual to try and find the reason why it's only providing a single connection at a time, I went back into the all too familiar `Startup.cs` to change the dependency lifetimes back to how they were before, just to confirm changing the lifetimes didn't do anything. I glanced at the code block for registering the `DbContext` again:

```C# Code
var npgsqlConnection = new NpgsqlConnection(connectionString);
services.AddDbContext<MyDbContext>(options => options.UseNpgsql(
    npgsqlConnection,
    options => options.MigrationsAssembly("Project.Namespace")));
```
If you already caught the mistake when first looking at this snippet at the start of this post, good catch! I certainly didn't. If you didn't, take a look at this line specifically:

```C# Code
var npgsqlConnection = new NpgsqlConnection(connectionString);
```

> Son of a-

I shook my head in disbelief. Then I laughed at myself. We were explicitly creating *one and only one* connection, and then told all `DbContext` instances to use exactly that one. No surprise that multiple requests at the same time would fail! It also explained why at least one request always worked. The requests were sent roughly simultaneously, so one of them would "win" and get to use the single available connection. The other ones would fail because the connection is busy. Sometimes requests were processed fast enough that the connection was available again for a second request, but it was never fast enough to handle three requests at roughly the same time.

The fix was easy:

```C# Code
services.AddDbContext<MyDbContext>(options => options.UseNpgsql(
    connectionString,
    options => options.MigrationsAssembly("Project.Namespace")));
```

Passing the connection string instead of an already created connection allowed ASP.NET to open connections as needed and everything worked smoothly. After refreshing the frontend roughly 100 times to confirm this bug was definitely not happening anymore, I was done.

A day of mine and a couple hours of other developers' time spent. Two lines changed. Problem fixed... At least I learned some things about ASP.NET because of it.

The only remaining mystery was why this popped up only now. I was sure that I had multiple requests in parallel before while testing, and that worked without any issues. I took a look at the git history of `Startup.cs` and found that this bug was introduced when we changed how the application was deployed, since the deployed service needed to construct its connection string dynamically and perform some additional authentication, none of which was needed when running locally. This was around the same time that the favorites feature was implemented, so the unfortunate timing of this bug made it look like the two were related.

## TL;DR

I encountered a bug with a straightforward cause and solution. However, I looked up causes and solutions online before trying to understand the problem myself. This led to wasting a lot of my and others' time because I tried a lot of things that I could have known to not solve the issue.

## The Lesson(s)

There are multiple things that I think I should have done sooner, which would have saved me a lot of time:

1. **Understand the issue before looking up solutions**. I did not *exactly* know what was happening when some of the requests failed, instead I looked up the error messages. The solutions I found looked reasonable enough, but if I understood the problem better I would have known that these didn't apply to my particular situation.
2. **Check the simple stuff first**. Even though most of the time a bug will not be in the most obvious and straightforward place, I should have still checked those places first. It does not take a lot of time in case the bug isn't there, but if it is, it will save you a lot of effort!
3. **Challenge assumptions sooner**. I spent way too long trying different dependency lifetimes instead of just thinking about whether they actually could be responsible for the issue. This goes hand-in-hand with lesson #1, since the suggested solutions online steered me into this direction.
4. **Don't rely on "magic" to work**. If I had known how dependency lifetimes worked beforehand, I could have applied lesson #3 a lot sooner.
5. **Narrow down when a bug was introduced**. This is a tricky one, because most of the time you can't narrow down the bug to the exact commit before you find the cause. However, if I had taken a look and saw what was happening around the time the favorites feature was added to the project, I would also have spotted the deployment-related changes and maybe looked into them.

With a bit of luck, I'll even remember these lessons next time I'm fixing bugs. Hopefully, these are helpful to you too, and you can apply them the next time you are stuck with some seemingly inexplicable issue!