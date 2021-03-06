---
layout: post
title:  "Caching in Neos: examples and gothchas"
tags: neos typoscript caching
comments: true
---

This week I finally learned how caching mechanisms work in [Neos](http://neos.typo3.org). Time to share!

##Basic concepts

A quick summary of what we know from the [official documentation](http://docs.typo3.org/neos/TYPO3NeosDocumentation/IntegratorGuide/ContentCache.html).

- Each TypoSript path may have its own caching configuration of type `cached`, `embed` and `uncached`. By default all paths are 'embedded' into their paraent's cache entry.
- For configuration of type `cached` there are two things you need to configure: what defines a unique cache entry (`entryIdentifier`) and in what circumstances it should be flushed (`entryTags`).
- For configuration of type `uncached` you must not forget to fill the `context` configuration with all of the context variables needed for rendering given path (like `node` or `documentNode`).


##Some examples

So in most cases you don't need to touch the cache configuration at all. Usually you would define new cache entries for the sake of flushing the cache for content that is comming from other pages, so the `entryTags` configuration would be closely related to the FlowQuery you use to pull in your content. There are three main options you have for configuring entry Tags: `NodeType_[My.Package:NodeTypeName]`, `Node_[Identifier]` and `DescendantOf_[Identifier]`.

Here are a few examples from our in-house news package:


###Flush the cache when a child of a certain node changes

With this FlowQuery I pull in news from a certain, dynamically configured node: `collection = ${q(rootNode).children('[instanceof Sfi.News:Listable]').get()}`

And here's the corresponding entryTag: `${'DescendantOf_' + rootNode.identifier}`.

###Flush the cache when a node of a certain type changes

With this FlowQuery we pull-in only inportant news of a special type to the carousel on the main page: `collection = ${q(site).find('[instanceof Sfi.News:ImportantMixin]').filter('[important = TRUE]').get()}`

And here's the corresponding entryTag: `${'NodeType_Sfi.News:ImportantMixin'}`

So far things look really easy, but wait, there are a few nuances that would make you scratch your head.


##Gotchas

###ContentCollection inside of Content

Neos has this little piece of configuration that caused me a lot of trouble:

{% highlight bash%}
prototype(TYPO3.Neos:Content) {
	prototype(TYPO3.Neos:ContentCollection) {
		# Make ContentCollection inside content node types embedded by default.
		# Usually there's no need for a separate cache entry for container content elements, but depending on the use-case
		# the mode can safely be switched to 'cached'.
		@cache {
			mode = 'embed'
		}
	}
}
{% endhighlight %}

Most people render their root content collection from their page object (or any object inheriting from TYPO3.Neos:Document), but not me. I always have some proxy object, inheriting from TYPO3.Neos:Content, inside of which I render the root content collection. The result? -- ContentCollections get embeded into the page object, so if I change some element inside of that ContentCollection and refresh the page, the changes are gone. Here's how to fix it:

{% highlight bash%}
prototype(TYPO3.Neos:Content).prototype(TYPO3.Neos:ContentCollection).@cache.mode = 'cached'
{% endhighlight %}


###Pagination

When using the typo3cr pagination widget, the state of the page is defined not only by the TypoScript path, but also by the pagination argument in request, so the `entryIdentifier` must include: `pagination = ${request.pluginArguments.YOUR_PAGINATION_WIDGET_ID.currentPage}`. Actually this is one of the rare cases where you need to add something non-standard to `entryIdentifier`. However, guess what, this config wouldn't work! The reason is given in the small note in the documentation, and it's easy to miss:

> In the cache hierarchy the outermost cache entry determines all the nested entries, so it's important to add values that influence the rendering for every cached path along the hierarchy.

Now we know how to make it work, add the same config to all parent cache definitions:

{% highlight bash%}
page.path.to.your.objct.@cache.entryIdentifier.pagination = ${request.pluginArguments.YOUR_PAGINATION_WIDGET_ID.currentPage}
page.@cache.entryIdentifier.pagination = ${request.pluginArguments.YOUR_PAGINATION_WIDGET_ID.currentPage}
root.@cache.entryIdentifier.pagination = ${request.pluginArguments.YOUR_PAGINATION_WIDGET_ID.currentPage}
{% endhighlight %}

So the rule is, _if you add something non-standard to an entryIdentifier, you must also include it in all parent cache defintions_.


###Caching time-dependant nodes (maximumLifetime)

Now there's a `maximumLifetime` setting, it lets you define a lifetime of a cache entry. At first I used to just set maximumLifetime to smaller value, but that never really pleased me. But you can use the full power of EEL in caching configuration, so why not try do something a bit more clever?

First of all, there's a [`cacheLifetime` FlowQuery operation](http://docs.typo3.org/neos/TYPO3NeosDocumentation/Appendixes/FlowQueryOperationReference.html#cachelifetime), accordion to the docs it will get the minimum of all allowed cache lifetimes for the nodes in the current FlowQuery context. This means it will evaluate to the nearest future value of the hiddenBeforeDateTime or hiddenAfterDateTime properties of all nodes in the context. So if you rely on hiddenBefore/AfterDateTime, all you probably need is something like this: `maximumLifetime = ${q(node).context({'invisibleContentShown': true}).children().cacheLifetime()}`.

But I was unfortunate enough to be using custom date property to calculate the visibility of my announcements, so here's how I calculated the correct lifetime value myself: `maximumLifetime = ${q(collection).get(0).properties.date.timestamp - Date.now().timestamp}`. See [here for details](https://github.com/sfi-ru/Sfi.News/commit/f376bcf15abd6a6d7feceb9d47ec0a15ee4f1bd7).

##Use faster caching backends

In Flow you can use alternative type of caching backend for every type of cache. By default Flow and Neos use FileBackend, and it's supper slow with flushing tagged caches. For production sites it's an absolute must to use more advanced caching backends like Redis for tagged caches. It only takes a few minutes to adjust the configuration (you [do use Docker](http://dimaip.github.io/2015/03/03/hybrid-deploy-with-docker-and-surf/), right?) Here's how the [Caches.yaml](https://github.com/sfi-ru/SfiDistr/blob/master/Configuration/Production/Caches.yaml) should look. 

##More cool stuff coming

In the Neos 2.0 release there'll be a [very useful addition called GlobalCacheIdentifiers](https://review.typo3.org/#/c/36210/). Basically it stores all global things which influnce the rendering of a node, acting as default value for a nodeIdentifier: by default `format` and `baseUri`, but you can add more things there. So in Neos 2.0 we would no longer need to specify `entryIdentifier` in most of the cases.

More additions are comming in the future, like [runtime cached segments](https://review.typo3.org/#/c/36239/), but even what we have now if pretty awesome.

I hope this post has helped you to quickly get up to speed with caching concepts in Neos. If not, write a caching-related question in the comments and I'll try to help.

**Big thanks to Aske Ertmann and Christian Müller for help in understanding all this stuff and to Florian Weiss for proofreading!**
