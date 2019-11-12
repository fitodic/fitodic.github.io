---
layout: post
title: "Optimizing websites and APIs with cache and ETags"
date: 2019-11-12 17:20:58 +0100
categories: web
tags: web http cache
permalink: optimizing-websites-and-apis-with-cache-and-etags
---

When creating a public website or a web app with a public API, you generally want things to run as smooth as possible for the end-user. You also might want to minimize the server's response time and resource consumption and you want to do it as fast as possible.

Why would you do that? For any number of reasons, but most often as your site's traffic increases, it becomes a necessity to speed it up as much as possible.

How would you do that? Performance benchmarks, when done properly, oftentimes tell you where you should look. At first, most people focus on tuning database queries or revisit some of the tools or libraries they are using. Although these steps are important and have their merits, that's not why we are here. We are here because you want to get the most bang for your buck, especially if you have a website or API that's read a lot. That's where [cache](#cache) and [`ETag`](#etag) come in.

Before we dive right in, I should tell you that when it comes to caching and `ETag`s, there is no silver bullet. If someone tells you otherwise, for instance that there is a service or an approach that magically makes all your problems go away, you can safely assume their set of experiences and situations they have dealt with is most probably rather limited. There's nothing wrong with that, just be aware that their solutions might not be applicable to your use case.

Why am I telling you this? Because I've had this talk with several colleagues and noticed that I was repeating the same story and sending the same references all over again. This is my first attempt at scaling this process. I don't have all the answers and there are certainly things that I am not aware of at the moment, but I'm going to outline the things I wish someone had told me when I was first starting out in this direction, i.e. optimizing web sites and APIs for high intensity traffic.

## Cache

Have you noticed how good computers are at repetitive tasks? Well, it turns out even computers have their limits. This limit depends mostly on the amount of "juice" your machine has at its disposal, but ask yourself this: **why would you want to execute the same query/computation/operation, over and over again, for each user, only to get the same result?**

When a user makes a request to a specific resource, for instance its profile, What the server or application generally does is it computes the result or retrieves the data from the database, serializes it into the requested format and sends it back to the user. Without cache, this process is repeated for each request, over and over again.

There has to be a better way, right? Regardless of the amount of users that are triggering this event, why would you want to waste precious resources on something like that? Wouldn't these resources be better spent on something else? Or not spent at all? Is there a better way?

As it turns out, there is. Consider the case we've described above. If we use a caching mechanism, the server will save the result (retrieved or serialized data) into memory before sending it to the user who has made the request. That way, if the same resource is requested again, instead of retrieving or computing the result again, the end-result will be ready. It will be retrieved from memory and sent back to the user almost instantly.

### Services that provide cache

Sound great, but you may be asking yourself how can you implement it? If you are using a database, such as PostgreSQL, you are already using it via [`shared_buffers`](https://www.postgresql.org/docs/current/runtime-config-resource.html#GUC-SHARED-BUFFERS). This setting tells PostgreSQL the amount of memory it has at its disposal for caching data. The defaults are pretty conservative, as are some of its other settings, so you may want to [tune it](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server), depending on your needs and available resources.

Apart from the database itself, there are services that are dedicated to caching data. [`memcached`](https://www.memcached.org/) and [`redis`](https://redis.io/) are two popular options, but there are many more. They are oftentimes key-value storages where you can store the results of database calls, results of computations or even entire rendered pages. Most programming languages and Web frameworks have libraries and interfaces to interact with them so you don't have to reinvent the wheel. For example, [Django has an entire section dedicated to cache](https://docs.djangoproject.com/en/dev/topics/cache/).

### HTTP cache

When working with the Web, it is not enough to cache your own data. There are other forces at work here outside of your control, which are sometimes called ["downstream" caches](https://docs.djangoproject.com/en/dev/topics/cache/#downstream-caches), such as Internet Service Providers (ISP), cache proxies or even the Web browsers themselves. These caches may come between the end-user and your application, and it is important to be aware of them, especially when dealing with private data.

To be on the safe side, you should send HTTP headers that define the resource's cache policy. The two of most important headers are [`Cache-Control`](cache-control) and [`Vary`](vary). [`Cache-Control`](cache-control) defines whether or not the response should be cached by the client, for how long and when should it be revalidated, whereas the `Vary` header indicates which HTTP headers are used when selecting a representation of the resource. If you are using a Web framework, it probably has its own mechanisms for generating these headers. For example, Django has decorators for [the `Vary` HTTP header](https://docs.djangoproject.com/en/dev/topics/cache/#using-vary-headers), as well as for the [`Cache-Control` HTTP header](https://docs.djangoproject.com/en/dev/topics/cache/#controlling-cache-using-other-headers).

As a side note, if you are using `SessionAuthenticationMiddleware`, [each response will have a `Vary: Cookie` response header](https://code.djangoproject.com/ticket/23939).

### Cache expiration

> There are only two hard things in Computer Science: cache invalidation and naming things.
> -- Phil Karlton

Since Web frameworks provide most of the mechanisms to deal with cache, both internally (via their APIs and interfaces) and externally (via HTTP headers). That means that the hardest part is left to the developer: **how to invalidate cache so that fresh data is sent as soon as possible?**

Theoretically, data can be stored in- and be served from cache indefinitely, but you don't want that. You want to evict old and rarely used data, but what about data that is updated?

Once a resource is updated on the server, the previous version of the same resource stored in cache should be either evicted, invalidated or updated in order to server the new, updated version. Which path should you choose? It's basically up to you, i.e. your application and use-case, but there are several options that come to mind:

1. You can add a `last_modified` field to each resource (for simplicity's sake let's assume a resource represents a row in a database table) that it updated on each save, and use the resource's primary key (e.g. its ID) and the `last_modified` field to construct a cache key and use it to save to and retrieved the data from cache. This is a relatively simple approach that is widely applicable, however it does have the overhead of retrieving the object from the database, at the very least its ID and `last_modified` values to construct the cache key, before retrieving the cached data. Worst case scenario is that if the cache is empty, two database queries will be executed before the data is stored in cache for subsequent requests.

2. The first approach could be further optimized by caching all database queries, using something like [`django-cachalot`](https://django-cachalot.readthedocs.io/en/latest/) which [caches the results of database queries and clears them once the data in the database table is changed](https://django-cachalot.readthedocs.io/en/latest/how.html#monkey-patching). However, this approach does have its limitations, [primarily when there is a high amount of database modifications](https://django-cachalot.readthedocs.io/en/latest/limits.html#high-rate-of-database-modifications).

3. To avoid the aforementioned issues, you could implement something like [_per-site_](https://docs.djangoproject.com/en/dev/topics/cache/#the-per-site-cache) or [_per-view_](https://docs.djangoproject.com/en/dev/topics/cache/#the-per-view-cache) cache, but a vanilla implementation such as this means your application would be serving stale data until the previous version expires. This could be alleviated by passing a cache key constructor to `drf-extensions`' [`cache_response` decorator](https://chibisov.github.io/drf-extensions/docs/#cache-key), but that oftentimes raises the issues mentioned in point 1.

4. If you want fine-grained control, you can always [cache response fragments](https://docs.djangoproject.com/en/dev/topics/cache/#template-fragment-caching), but that is susceptible to nitpicking and inadvertently caching block that should not be cached (e.g. user-specific data).

5. There is also [cache versioning](https://docs.djangoproject.com/en/dev/topics/cache/#cache-versioning). Basically, each time a resource is updated, the cache key's version is incremented. Although this means that only the latest version of the resource is cached, the downside is that you should keep tabs on version numbers.

6. Finally, you can always use the [low-level API](https://docs.djangoproject.com/en/dev/topics/cache/#the-low-level-cache-api), i.e. implementing the cache invalidation logic yourself. For example, each time a resource is saved, you would construct the resource's cache key, try to locate it, and if it doesn't exist, create it. If it does, delete it and set it anew.

As I said before, the path you choose to go is entirely up to you. There is nothing wrong with combining all of the aforementioned approaches. As always, the most important thing to do is to choose the right tools for the job. These are just pointers with useful links so you know what are your options are and where to start. The end result depends primarily on your application and usage patterns.

## ETag

In the [HTTP cache](#http-cache) section I mentioned "downstream" caches and the role that `Cache-Control` and `Vary` headers play at informing proxy caches and other intermediaries about the resource's _freshness_. As it turns out, there is another set of HTTP headers that helps clients (e.g. Web browsers) determine whether or not the received resource is still fresh, and those headers are [`ETag`](etag), [`If-Match`](if-match) and [`If-None-Match`](if-none-match).

The [`ETag`](etag) HTTP response header is as a resource identifier, or better yet, an identifier of a particular version of the resource. Sort of like the resource's fingerprint that changes whenever the resource is updated. This identifier usually comes in the form of a hash, generated by the server. The algorithm that's used to generate it primarily depends on the resource it is calculating the `ETag` for, but there are a few common use-cases:

* If the resource is an entry in the database, you can use the database table's name (including the schema if necessary), the entry's primary key and the `last_modified` field (if available);
* If the resource is an HTML page composed of various different resources, you can use the entire HTML representation as input;
* If the resource is a file on disk, the file's modification time and size can be used as inputs.

You may be asking yourself how does this help the cache process? Why go through all this trouble of generating the `ETag` just to match it with the resource it came with? There are two reasons you would want to do that:

1. **It tells the client that sent the request (e.g. Web browser) to cache the response and keep reusing it until the `ETag` changes, thus saving bandwidth.**
2. **It prevents the client from updating the resource if the resource has been updated in the meantime.**

How is this achievable? By sending conditional requests.

### Conditional requests

Sending either [`If-Match`](if-match) or [`If-None-Match`](if-none-match) header in the request makes the request conditional, but when should you use which? The good news is that browsers do most of it for you by default, but here are the details behind these types of requests.

When retrieving data, the Web browser requests the resource from the server and receives the resource's representation along with a couple of HTTP headers, among which is the `ETag` header. The `ETag`'s value is used for retrieving the cached resource from the browser's cache. After a while, the resource needs to be revalidated (depending on the expiration time set by the `Cache-Control` HTTP header), so the browser requests the resource from the server (again), but this time, it adds the `ETag`'s value in the `If-None-Match` HTTP header. This tells the server to check whether or not the resource has changed since the last request. The server generates the resource's `ETag` and if the two values match, the resource has not changed, and the server returns a [`304 Not Modified`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/304) response with an empty body, telling the browser to use the cached resource. Otherwise, the server returns a [`200 OK`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200) response with the resource's new representation and a new `ETag` value. **What's the benefit of this process? You save on bandwidth by not receiving the data you already have stored locally.**

That covers data retrieval, but the `ETag` header is also useful when updating resources. In fact, it helps prevent simultaneous updates of the same resource from overwriting each other, or ["mid-air collisions"](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag#Caching_of_unchanged_resources). How would you do that? Let's say you've retrieved a resource and edited it in your browser. You click the _Save_ button, and the browser sends a PUT request with `ETag`'s value in the `If-Match` HTTP header. If the resource has changed in the meantime, the server will respond with a [`412 Precondition Failed`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/412) response, thus **preventing you from overriding the resource that has been updated while you were editing it.**


## Conclusion

Caching is a broad subject and this post is intended to give you a high-level overview of its application in Web development. This post includes a couple of examples, but there are certainly many more, from code snippets to HTTP headers that can be used as well (e.g. [`Last-Modified`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Last-Modified)). If you want to get into more details, there are references to other resources that go into more practical details, especially from the [Mozilla Developer Network](https://developer.mozilla.org/en-US/) and the [Django](https://www.djangoproject.com/) Web framework, both of them being excellent sources for further reading.

There are

## Resources

* [MDN: HTTP cache](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
* [Django: Cache](https://docs.djangoproject.com/en/dev/topics/cache/)
* PostgreSQL: [Docs](https://www.postgresql.org/docs/) and [Wiki](https://wiki.postgresql.org/wiki/Main_Page)

[etag]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag
[if-match]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Match
[if-none-match]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-None-Match
[cache-control]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control
[vary]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary
