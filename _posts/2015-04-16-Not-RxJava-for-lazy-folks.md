---
layout: post
title:  "NotRxJava guide for lazy folks"
date:   2015-04-16
categories: RxJava Android 
permalink: /NotRxJava/
comments: true
---

These days if you are an android developer, you might hear some hype about RxJava. 
RxJava is library which can help you get rid of all you complex write-only code that deals with asynchronous events. Once you start using it in your project -- you will use it everywhere.

The main pitfall here is steep learning curve. If you have never used RxJava before, it will be hard or confusing to take full advantage of it for the first time. The whole way you think about writing code is a little different.
Such learning curve creates problems for massive RxJava adoption in most projects.

Of course there are a lot of tutorials and code examples around that explain how to use RxJava. 
Developer interested in learning and using RxJava can first visit  the official [Wiki](https://github.com/ReactiveX/RxJava/wiki) that contains great explanation of what Observable is, how it's related to Iterable and Future. Another useful resource is [How To Use RxJava](https://github.com/ReactiveX/RxJava/wiki/How-To-Use-RxJava) page which shows code examples of how to emit items and `println` them.

But what one really wants to know, is what problem RxJava will solve and how it will help organize async code without actually learning what Observable is.

My goal here is to show some "prequel" to read before the official documentation in order to better understand the problems that RxJava tries to solve.
This article is positioned as a small walk-through on how to reorganize messy Async code by ourselfs to see how we can implement basic principles of RxJava without actually using RxJava.

So If you are still curious let's get started!

## Cat App
So let's create a *real world* example. So we know that cats are the engine of technology progress, so let's build 
a typical app for downloading cat pictures.

#### So here is the task:
> We have a webservice that provides api to search the whole internet for
> images of cats by given query. Every image will contain cuteness
> parameter - integer value that describes how cute is that picture. Our
> task will be download a list of cats, choose the most cutest, and save
> it to local storage.

We will focus only on downloading, processing and saving cats data. 

So let's start:

## Model and API

Here is a simple data structure that will describe our 'Cat'

```java 
public class Cat implements Comparable<Cat>{
    Bitmap image;
    int cuteness;

    @Override
    public int compareTo(Cat another) {
        return Integer.compare(cuteness, another.cuteness);
    }
}
```

And our api calls that we will use from cat-sdk.jar will be in good-old blocking style.

```java
public interface Api {
    List<Cat> queryCats(String query);
    Uri store(Cat cat);
}
```

Heh, seems clear yet? Sure! Let's write client business logic.

```java
public class CatsHelper {

    Api api;

    public Uri saveTheCutestCat(String query){
        List<Cat> cats = api.queryCats(query);
        Cat cutest = findCutest(cats);
        Uri savedUri = api.store(cutest);
        return savedUri;
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

Damn, so clear and so simple. Let's think a bit about how cool this code is.
The main method saveTheCutestCat contains only 3 functions.
Take a few minutes to look at code and think about functions.
This fundamental feature of our language is so amazing.
You take functions, feed the input, and receive the output of it.
And we wait while function is doing its work.

So simple and yet so powerful. Let's think about another advantages of simple functions:

### Composition
As you can see we created one function (`saveTheCutestCat`) from 3 others, thus we composed them.
Any of those functions is like LEGO blocks: we use them to connect one to another to get _composed_ LEGO block (which actually can be composed later).
Composing functions is so simple -- just feed result of one function as argument to another in right order, what could be simpler?

### Error propagation
Another good thing about functions is the way we deal with errors. Any function can terminate its execution with error, in java we call it *throwing an Exception*. And this error can be handled on any level.
In java we do this with *try/catch* block.
The key point here is that we don't need to handle errors for every function, but we can handle *all* possible errors for the whole block of code. Like:

```java
try{
    List<Cat> cats = api.queryCats(query);
    Cat cutest = findCutest(cats);
    Uri savedUri = api.store(cutest);
    return savedUri;
} catch (Exception e) {
    e.printStackTrace()
    return someDefaultValue;
}
```
In this case we will handle any errors that happen during the execution.
Or we can propagate it to the next level, if we just leave our code without *try/catch* block.

## Go Async

But we live in a world where we cannot wait. Sometimes it's not possible to have only blocking calls. And actually in android you always need to deal with asynchronous code.

Take for example Android’s default `OnClickListener`. When you want to handle a view click, you must provide a listener (callback) which will be invoked when user clicks. And there is no reasonable way to receive clicks in a blocking manner. So clicks are always asynchronous. 
So let's try to deal with async code now.

#### Async network call
Let's now imagine that to make a query we use some non-blocking HTTP client (like [Ion](https://github.com/koush/ion)). 
So imagine our `cats-sdk.jar` has updated and replaced it's API with async calls.

So new Api interface:

```java
public interface Api {
    interface CatsQueryCallback {
        void onCatListReceived(List<Cat> cats);
        void onError(Exception e);
    }


    void queryCats(String query, CatsQueryCallback catsQueryCallback);

    Uri store(Cat cat);
}
```

So now we have async cats list fetching, and to receive the result there is additional callback `CatsQueryCallback` which can return a list of cats or an error.

So as api have changed we now have to change our `CatsHelper`:

```java
public class CatsHelper {

    public interface CutestCatCallback {
        void onCutestCatSaved(Uri uri);
        void onQueryFailed(Exception e);
    }

    Api api;

    public void saveTheCutestCat(String query, CutestCatCallback cutestCatCallback){
        api.queryCats(query, new Api.CatsQueryCallback() {
            @Override
            public void onCatListReceived(List<Cat> cats) {
                Cat cutest = findCutest(cats);
                Uri savedUri = api.store(cutest);
                cutestCatCallback.onCutestCatSaved(savedUri);
            }

            @Override
            public void onQueryFailed(Exception e) {
                cutestCatCallback.onError(e);
            }
        });
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

As now we can't use the blocking api, we can not write our client as a blocking call (actually we can, but with explicit thread blocking with `synchronized`, `CountdownLatch`, etc, but we need to deal with async underlying operations anyway).
So we can't write our `saveTheCutestCat` function to return a value, we need to introduce callback to handle the result in async way.

Let's go even further, what if two of our API calls are async, for example we are using non-blocking  IO to write into a file.

```java
public interface Api {
    interface CatsQueryCallback {
        void onCatListReceived(List<Cat> cats);
        void onQueryFailed(Exception e);
    }

    interface StoreCallback{
        void onCatStored(Uri uri);
        void onStoreFailed(Exception e);
    }


    void queryCats(String query, CatsQueryCallback catsQueryCallback);

    void store(Cat cat, StoreCallback storeCallback);
}
```

And our helper:

```java
public class CatsHelper {

    public interface CutestCatCallback {
        void onCutestCatSaved(Uri uri);
        void onError(Exception e);
    }

    Api api;

    public void saveTheCutestCat(String query, CutestCatCallback cutestCatCallback){
        api.queryCats(query, new Api.CatsQueryCallback() {
            @Override
            public void onCatListReceived(List<Cat> cats) {
                Cat cutest = findCutest(cats);
                api.store(cutest, new Api.StoreCallback() {
                    @Override
                    public void onCatStored(Uri uri) {
                        cutestCatCallback.onCutestCatSaved(uri);
                    }

                    @Override
                    public void onStoreFailed(Exception e) {
                        cutestCatCallback.onError(e);
                    }
                });
            }

            @Override
            public void onQueryFailed(Exception e) {
                cutestCatCallback.onError(e);
            }
        });
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

And just look at the code once again. Is it as pretty as before? No! It's awful.
Now it has much more code noise, more curly braces, but the logic is the same.

What about composition? It's gone!
Now you can't just compose operations as before. On any async operation you must create callback for it and insert continuation code inside it. Manually. Every time. 

How about error propagation? How about **No!**
In this code errors are not propagated automatically, we need to re-pass it further by ourselves (see `onStoreFailed` and `onQueryFailed` methods).

Is it harder to read this code and to find possible bugs? Definitely!

### The end?

So what? what can we do with it? Are we trapped in this world of non-composable callback hell?
Fasten your seat belts, we will try to fix!

## To the better world!

### Generic callback

If we look closer to our Callback interfaces we can spot the common pattern:

1. All of them have a method that delivers the result (`onCutestCatSaved`, `onCatListReceived`, `onCatStored`)
2. Most of them (all in our case) have a method to handle errors that happen during operation (`onError`, `onQueryFailed`, `onStoreFailed`)

So we can create a generic callback to replace them all.
But as we can not change api call signatures we would have to create wrapped calls.

So our generic callback will looks like:

```java
public interface Callback<T> {
    void onResult(T result);
    void onError(Exception e);
}
```

And let's just create `ApiWrapper` to change signature of our calls:

```java
public class ApiWrapper {
    Api api;

    public void queryCats(String query, Callback<List<Cat>> catsCallback){
        api.queryCats(query, new Api.CatsQueryCallback() {
            @Override
            public void onCatListReceived(List<Cat> cats) {
                catsCallback.onResult(cats);
            }

            @Override
            public void onQueryFailed(Exception e) {
                catsCallback.onError(e);
            }
        });
    }

    public void store(Cat cat, Callback<Uri> uriCallback){
        api.store(cat, new Api.StoreCallback() {
            @Override
            public void onCatStored(Uri uri) {
                uriCallback.onResult(uri);
            }

            @Override
            public void onStoreFailed(Exception e) {
                uriCallback.onError(e);
            }
        });
    }
}
```

So it's just dummy logic to propagate resuts/errors to our new `Callback`

And finally our `CatsHelper`:

```java
public class CatsHelper{

    ApiWrapper apiWrapper;

    public void saveTheCutestCat(String query, Callback<Uri> cutestCatCallback){
        apiWrapper.queryCats(query, new Callback<List<Cat>>() {
            @Override
            public void onResult(List<Cat> cats) {
                Cat cutest = findCutest(cats);
                apiWrapper.store(cutest, cutestCatCallback);
            }

            @Override
            public void onError(Exception e) {
                cutestCatCallback.onError(e);
            }
        });
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

Ok, little bit smaller than previous one. We could decrease the level of callbacks by one here by feeding our top `cutestCatCallback` directly to the `apiWrapper.store` call, as signatures of the callbacks are the same.

But can we do even better? Sure! 

### You gotta keep it separated

Let's look now on our async operations(`queryCats`, `store` and resulting `saveTheCutestCat`).
All of them follow the same pattern. The method that invokes them has some valuable arguments (`query`, `cat`) and a callback object as one of the arguments. 
Once more: Any async operation takes all the regular arguments and additionally a callback. What if we try to separate these stages, so that every async operation will take only valuable arguments, and then return some temporary object that will take callback.

Let's try to apply this pattern and look if it's helpful.

So If we will return temporary objects for async operation, we need to declare one. As this object is almost follows the common behavior (taking callback as single argument) we will define this class for all our async operations.
Let's call it `AsyncJob`.
>P.S. I would rather call it `AsyncTask`, but I don't want to confuse you with another existing abstraction (which also happens to be bad) over async operations.

So:

```java
public abstract class AsyncJob<T> {
    public abstract void start(Callback<T> callback);
}
```
Pretty simple here. Method start will take `Callback` and start it's *task*.

And changed wrapped API calls:

```java
public class ApiWrapper {
    Api api;

    public AsyncJob<List<Cat>> queryCats(String query) {
        return new AsyncJob<List<Cat>>() {
            @Override
            public void start(Callback<List<Cat>> catsCallback) {
                api.queryCats(query, new Api.CatsQueryCallback() {
                    @Override
                    public void onCatListReceived(List<Cat> cats) {
                        catsCallback.onResult(cats);
                    }

                    @Override
                    public void onQueryFailed(Exception e) {
                        catsCallback.onError(e);
                    }
                });
            }
        };
    }

    public AsyncJob<Uri> store(Cat cat) {
        return new AsyncJob<Uri>() {
            @Override
            public void start(Callback<Uri> uriCallback) {
                api.store(cat, new Api.StoreCallback() {
                    @Override
                    public void onCatStored(Uri uri) {
                        uriCallback.onResult(uri);
                    }

                    @Override
                    public void onStoreFailed(Exception e) {
                        uriCallback.onError(e);
                    }
                });
            }
        };
    }
}
```

So far so good. We just applied partial application to our wrapped API calls. Now we can operate with `AsyncJob` to start it whenever we want. So it is time to change `CatsHelper`.

```java
public class CatsHelper {

    ApiWrapper apiWrapper;

    public AsyncJob<Uri> saveTheCutestCat(String query) {
        return new AsyncJob<Uri>() {
            @Override
            public void start(Callback<Uri> cutestCatCallback) {
                apiWrapper.queryCats(query)
                        .start(new Callback<List<Cat>>() {
                            @Override
                            public void onResult(List<Cat> cats) {
                                Cat cutest = findCutest(cats);
                                apiWrapper.store(cutest)
                                        .start(new Callback<Uri>() {
                                            @Override
                                            public void onResult(Uri result) {
                                                cutestCatCallback.onResult(result);
                                            }

                                            @Override
                                            public void onError(Exception e) {
                                                cutestCatCallback.onError(e);
                                            }
                                        });
                            }

                            @Override
                            public void onError(Exception e) {
                                cutestCatCallback.onError(e);
                            }
                        });
            }
        };
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

Wow, the previous version was simpler. What benefits do we have now? -- Now we actually return `AsyncJob<Uri>` of 'composed' operations to the client side. So a client (in activity or fragment) can operate with the composed job.

Code is even uglier now, but we will fix it, so...

### Breaking things

Here is our happy-path dataflow here:

```
         (async)                 (sync)           (async)
query ===========> List<Cat> -------------> Cat ==========> Uri
       queryCats              findCutest           store
```

To make code readable again let's break this flow into these operations.
But one more thing, let's think about it little bit: if some operation is async, then any operation with it is async too,
Example: if querying cats is an async operation, then finding the cutest cat (even with single blocking call) is an async operation too (for the client that want to receive the result).

So we can divide our whole method into smaller operations with AsyncJobs:

```java
public class CatsHelper {

    ApiWrapper apiWrapper;

    public AsyncJob<Uri> saveTheCutestCat(String query) {
        AsyncJob<List<Cat>> catsListAsyncJob = apiWrapper.queryCats(query);
        AsyncJob<Cat> cutestCatAsyncJob = new AsyncJob<Cat>() {
            @Override
            public void start(Callback<Cat> callback) {
                catsListAsyncJob.start(new Callback<List<Cat>>() {
                    @Override
                    public void onResult(List<Cat> result) {
                        callback.onResult(findCutest(result));
                    }

                    @Override
                    public void onError(Exception e) {
                        callback.onError(e);
                    }
                });
            }
        };

        AsyncJob<Uri> storedUriAsyncJob = new AsyncJob<Uri>() {
            @Override
            public void start(Callback<Uri> cutestCatCallback) {
                cutestCatAsyncJob.start(new Callback<Cat>() {
                    @Override
                    public void onResult(Cat cutest) {
                        apiWrapper.store(cutest)
                                .start(new Callback<Uri>() {
                                    @Override
                                    public void onResult(Uri result) {
                                        cutestCatCallback.onResult(result);
                                    }

                                    @Override
                                    public void onError(Exception e) {
                                        cutestCatCallback.onError(e);
                                    }
                                });
                    }

                    @Override
                    public void onError(Exception e) {
                        cutestCatCallback.onError(e);
                    }
                });
            }
        };
        return storedUriAsyncJob;
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

Looks bigger, but cleaner. Lower level of nested callback, understandable variable names (`catsListAsyncJob`, `cutestCatAsyncJob`, `storedUriAsyncJob`)

Looks better now, but let's do something more:

### Simple Mapping

Now I want you to look at the part where we create `AsyncJob<Cat> cutestCatAsyncJob`:

```java
AsyncJob<Cat> cutestCatAsyncJob = new AsyncJob<Cat>() {
            @Override
            public void start(Callback<Cat> callback) {
                catsListAsyncJob.start(new Callback<List<Cat>>() {
                    @Override
                    public void onResult(List<Cat> result) {
                        callback.onResult(findCutest(result));
                    }

                    @Override
                    public void onError(Exception e) {
                        callback.onError(e);
                    }
                });
            }
        };
```

All  these 16 lines of code have only one useful operation (for our logic): 

```java
findCutest(result)
```
Remaining code is just boilerplate noise to start another `AsyncJob` and to propagate result and error.
Moreover, this noise isn't task-specific -- so we can actually move it somewhere to not distract us from the actual code.

So how could we write it? Two thing we need to have to implement this operation:

1. `AsyncJob` which result we will transform
2. Transforming function

Here is one more problem: as we can't pass functions directly in java, we need to do it with classes (and interfaces), so we need to define what is 'function':

```java
public interface Func<T, R> {
    R call(T t);
}
```
Pretty simple. `Func` interface have two type members `T` which corresponds to argument type and `R` which is result type.

As we transform the result from one `AsyncJob` we will do some kind of *mapping* between values, so let's call the function `map`. Good place to define this function is to make it an instance method of `AsyncJob` class, so we can access source `AsyncJob` as `this`:

```java
public abstract class AsyncJob<T> {
    public abstract void start(Callback<T> callback);

    public <R> AsyncJob<R> map(Func<T, R> func){
        final AsyncJob<T> source = this;
        return new AsyncJob<R>() {
            @Override
            public void start(Callback<R> callback) {
                source.start(new Callback<T>() {
                    @Override
                    public void onResult(T result) {
                        R mapped = func.call(result);
                        callback.onResult(mapped);
                    }

                    @Override
                    public void onError(Exception e) {
                        callback.onError(e);
                    }
                });
            }
        };
    }
}
```

Cool, but how `CatsHelper` will look now:

```java
public class CatsHelper {

    ApiWrapper apiWrapper;

    public AsyncJob<Uri> saveTheCutestCat(String query) {
        AsyncJob<List<Cat>> catsListAsyncJob = apiWrapper.queryCats(query);
        AsyncJob<Cat> cutestCatAsyncJob = catsListAsyncJob.map(new Func<List<Cat>, Cat>() {
            @Override
            public Cat call(List<Cat> cats) {
                return findCutest(cats);
            }
        });

        AsyncJob<Uri> storedUriAsyncJob = new AsyncJob<Uri>() {
            @Override
            public void start(Callback<Uri> cutestCatCallback) {
                cutestCatAsyncJob.start(new Callback<Cat>() {
                    @Override
                    public void onResult(Cat cutest) {
                        apiWrapper.store(cutest)
                                .start(new Callback<Uri>() {
                                    @Override
                                    public void onResult(Uri result) {
                                        cutestCatCallback.onResult(result);
                                    }

                                    @Override
                                    public void onError(Exception e) {
                                        cutestCatCallback.onError(e);
                                    }
                                });
                    }

                    @Override
                    public void onError(Exception e) {
                        cutestCatCallback.onError(e);
                    }
                });
            }
        };
        return storedUriAsyncJob;
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

Much better. Creating `AsyncJob<Cat> cutestCatAsyncJob` is now takes 6 lines of code and only single level of callback.

### Advanced mapping

Cool stuff, but another part of creating `AsyncJob<Uri> storedUriAsyncJob` is still ugly. Can we apply map here? 
Let's give it a try:

```java
public class CatsHelper {

    ApiWrapper apiWrapper;

    public AsyncJob<Uri> saveTheCutestCat(String query) {
        AsyncJob<List<Cat>> catsListAsyncJob = apiWrapper.queryCats(query);
        AsyncJob<Cat> cutestCatAsyncJob = catsListAsyncJob.map(new Func<List<Cat>, Cat>() {
            @Override
            public Cat call(List<Cat> cats) {
                return findCutest(cats);
            }
        });

        AsyncJob<Uri> storedUriAsyncJob = cutestCatAsyncJob.map(new Func<Cat, Uri>() {
            @Override
            public Uri call(Cat cat) {
                return apiWrapper.store(cat);
        //      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ will not compile
        //      Incompatible types:
        //      Required: Uri
        //      Found: AsyncJob<Uri>                
            }
        });
        return storedUriAsyncJob;
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

Sigh... Not so easy, let's fix type of resulting variable, another try:

```java
public class CatsHelper {

    ApiWrapper apiWrapper;

    public AsyncJob<Uri> saveTheCutestCat(String query) {
        AsyncJob<List<Cat>> catsListAsyncJob = apiWrapper.queryCats(query);
        AsyncJob<Cat> cutestCatAsyncJob = catsListAsyncJob.map(new Func<List<Cat>, Cat>() {
            @Override
            public Cat call(List<Cat> cats) {
                return findCutest(cats);
            }
        });

        AsyncJob<AsyncJob<Uri>> storedUriAsyncJob = cutestCatAsyncJob.map(new Func<Cat, AsyncJob<Uri>>() {
            @Override
            public AsyncJob<Uri> call(Cat cat) {
                return apiWrapper.store(cat);
            }
        });
        return storedUriAsyncJob;
        //^^^^^^^^^^^^^^^^^^^^^^^ will not compile
        //      Incompatible types:
        //      Required: AsyncJob<Uri>
        //      Found: AsyncJob<AsyncJob<Uri>>
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```
We can only have `AsyncJob<AsyncJob<Uri>>` at this point. Do we need to go deeper?
What we want is to flatten one level of `AsyncJob` as two async operations are just another single async operation.

What we need now is to have map that will take function which returns not just `R` but `AsyncJob<R>`. 
This operation should behave like `map`, but in the end it should `flatten` our nested `AsyncJob`. 
Let's call it `flatMap` and try to implement it: 

```java
public abstract class AsyncJob<T> {
    public abstract void start(Callback<T> callback);

    public <R> AsyncJob<R> map(Func<T, R> func){
        final AsyncJob<T> source = this;
        return new AsyncJob<R>() {
            @Override
            public void start(Callback<R> callback) {
                source.start(new Callback<T>() {
                    @Override
                    public void onResult(T result) {
                        R mapped = func.call(result);
                        callback.onResult(mapped);
                    }

                    @Override
                    public void onError(Exception e) {
                        callback.onError(e);
                    }
                });
            }
        };
    }

    public <R> AsyncJob<R> flatMap(Func<T, AsyncJob<R>> func){
        final AsyncJob<T> source = this;
        return new AsyncJob<R>() {
            @Override
            public void start(Callback<R> callback) {
                source.start(new Callback<T>() {
                    @Override
                    public void onResult(T result) {
                        AsyncJob<R> mapped = func.call(result);
                        mapped.start(new Callback<R>() {
                            @Override
                            public void onResult(R result) {
                                callback.onResult(result);
                            }

                            @Override
                            public void onError(Exception e) {
                                callback.onError(e);
                            }
                        });
                    }

                    @Override
                    public void onError(Exception e) {
                        callback.onError(e);
                    }
                });
            }
        };
    }
}
```

Yeah, noisy implementation of flatMap, but all of it is in the same place, and we will not see it in our client code. So our fixed `CatsHelper`:

```java
public class CatsHelper {

    ApiWrapper apiWrapper;

    public AsyncJob<Uri> saveTheCutestCat(String query) {
        AsyncJob<List<Cat>> catsListAsyncJob = apiWrapper.queryCats(query);
        AsyncJob<Cat> cutestCatAsyncJob = catsListAsyncJob.map(new Func<List<Cat>, Cat>() {
            @Override
            public Cat call(List<Cat> cats) {
                return findCutest(cats);
            }
        });

        AsyncJob<Uri> storedUriAsyncJob = cutestCatAsyncJob.flatMap(new Func<Cat, AsyncJob<Uri>>() {
            @Override
            public AsyncJob<Uri> call(Cat cat) {
                return apiWrapper.store(cat);
            }
        });
        return storedUriAsyncJob;
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

Yeah! It works, and it's much more simpler to read and to write.

### Final point

Look at the resulting code again. Does it look familiar to you? Take one more look.
No? Will it be more easy if I will convert anonymous classes to java8 lambdas (just for better look, the logic is the same).

```java
public class CatsHelper {

    ApiWrapper apiWrapper;

    public AsyncJob<Uri> saveTheCutestCat(String query) {
        AsyncJob<List<Cat>> catsListAsyncJob = apiWrapper.queryCats(query);
        AsyncJob<Cat> cutestCatAsyncJob = catsListAsyncJob.map(cats -> findCutest(cats));
        AsyncJob<Uri> storedUriAsyncJob = cutestCatAsyncJob.flatMap(cat -> apiWrapper.store(cat));
        return storedUriAsyncJob;
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

Is it better now? I think this code looks very similar to our first blocking version:

```java
public class CatsHelper {

    Api api;

    public Uri saveTheCutestCat(String query){
        List<Cat> cats = api.queryCats(query);
        Cat cutest = findCutest(cats);
        Uri savedUri = api.store(cutest);
        return savedUri;
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

Yes! This is it! Logic is the same! And even more -- semantics are the same too.

Do we have composability? Heck yeah! 
We combined async operations and return composed one!

Error propagation? Sure! 
All errors will be passed directly to the final callback.

And finally...

##[RxJava](https://github.com/ReactiveX/RxJava)

Hey, you don't need to copy these classes into your current project.
Because we just implemented poorly written, non thread safe version of small part of RxJava.

There are only some differences:

* `AsyncJob<T>` is actually [`Observable<T>`](http://reactivex.io/documentation/observable.html) and it can deliver not just a single result but a sequence (possibly empty) of them.
* `Callback<T>` is [`Observer<T>`](http://reactivex.io/documentation/operators/subscribe.html)  and besides of methods `onNext(T t)`, `onError(Throwable t)` has method `onCompleted()` that will notify that the `Observable` it wraps finished emitting items (as it can emit a sequence of them)
* `abstract void start(Callback<T> callback)` corresponds to [`Subscription subscribe(final Observer<? super T> observer)`](http://reactivex.io/RxJava/javadoc/rx/Observable.html#subscribe(rx.Observer)) that also returns [`Subscription`](http://reactivex.io/RxJava/javadoc/rx/Subscription.html) which you can use to cancel receiving items, when you don't need  them anymore.
* Besides methods `map` and `flatMap` `Observable` has [additional](http://reactivex.io/documentation/operators.html) useful operations over Observalbes.

Here is an example how our code will look like with using RxJava:

```java
public class ApiWrapper {
    Api api;

    public Observable<List<Cat>> queryCats(final String query) {
        return Observable.create(new Observable.OnSubscribe<List<Cat>>() {
            @Override
            public void call(final Subscriber<? super List<Cat>> subscriber) {
                api.queryCats(query, new Api.CatsQueryCallback() {
                    @Override
                    public void onCatListReceived(List<Cat> cats) {
                        subscriber.onNext(cats);
                    }

                    @Override
                    public void onQueryFailed(Exception e) {
                        subscriber.onError(e);
                    }
                });
            }
        });
    }

    public Observable<Uri> store(final Cat cat) {
        return Observable.create(new Observable.OnSubscribe<Uri>() {
            @Override
            public void call(final Subscriber<? super Uri> subscriber) {
                api.store(cat, new Api.StoreCallback() {
                    @Override
                    public void onCatStored(Uri uri) {
                        subscriber.onNext(uri);
                    }

                    @Override
                    public void onStoreFailed(Exception e) {
                        subscriber.onError(e);
                    }
                });
            }
        });
    }
}

public class CatsHelper {

    ApiWrapper apiWrapper;

    public Observable<Uri> saveTheCutestCat(String query) {
        Observable<List<Cat>> catsListObservable = apiWrapper.queryCats(query);
        Observable<Cat> cutestCatObservable = catsListObservable.map(new Func1<List<Cat>, Cat>() {
            @Override
            public Cat call(List<Cat> cats) {
                return CatsHelper.this.findCutest(cats);
            }
        });
        Observable<Uri> storedUriObservable = cutestCatObservable.flatMap(new Func1<Cat, Observable<? extends Uri>>() {
            @Override
            public Observable<? extends Uri> call(Cat cat) {
                return apiWrapper.store(cat);
            }
        });
        return storedUriObservable;
    }

    private Cat findCutest(List<Cat> cats) {
        return Collections.max(cats);
    }
}
```

You can see that the code is the same except using `Observable` instead of `AsyncJob`


# Conclusion
 We see how with simple transformation we can create an abstraction over Async operations. This abstraction can be used to operate and compose async tasks just like simple functions. With this approach we can get rid of nested callbacks and hand-written error propagation during handling Async results.

If you still here I suggest you to relax, think a bit about duality of sync/async and watch this awesome [video](https://channel9.msdn.com/Events/Lang-NEXT/Lang-NEXT-2014/Keynote-Duality) from [Erik Meijer](http://en.wikipedia.org/wiki/Erik_Meijer_(computer_scientist)). 

### Useful Links

* [http://reactivex.io/]
* https://github.com/ReactiveX/RxJava
* https://github.com/ReactiveX/RxJava/wiki
* http://queue.acm.org/detail.cfm?id=2169076
* https://www.coursera.org/course/reactive
* http://blog.danlew.net/2014/09/15/grokking-rxjava-part-1/


### Acknowledgment
Thanks to my friend Alexander Yakushev for help with translation



