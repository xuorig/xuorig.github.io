---
title:  "A quick dive into how relay fetches GraphQL queries"
date:   2016-01-31 17:00:00
description: Quick intro to how Relay fetches GrapQL queries and caches it's results.
---

For my own curiosity I've recently decided to dive into Relay's source to understand
how it actually fetches queries and caches them. I thought I'd share the notes I took from my adventure into Relay.

*Dislaimer: this blog entry will be more like a note pad than a full blog.*

So I started At `RelayRenderer`. `RelayRenderer`'s goal is to render a container and it's query config after fulfilling its data dependencies. And this all basically starts at `componentWillReceiveProps`

### componentWillReceiveProps(nextProps)

#### _runQueries
calls `_runQueries` to update state
`_runQueries` will, as the name says, run the queries found in the `querySet`, retrieved by `getRelayQueries`.

#### getRelayQueries
`getRelayQueries` is called when rendering a Relay Container, and retrieves all queries for a component, given a route.
It caches using the route name and params.
`cacheKey = route.name + ':' + stableStringify(route.params);`


it will abort if there is a current pending request
if the forceFetch props is present: `RelayStore.forceFetch` is called with the `querySet`
if not `RelayStore.primeCache` is called with the `querySet`. Both `forceFetch` and `primeCache` are functions on `GraphQLQueryRunner`.


### GraphQLQueryRunner
This is the high-level entry point for sending queries to the GraphQL
endpoint. It provides methods for scheduling queries (`run`), force-fetching
queries (ie. ignoring the cache; `forceFetch`).
In order to send minimal queries and avoid re-retrieving data,
`GraphQLQueryRunner` maintains a registry of pending (in-flight) queries, and
"subtracts" those from any new queries that callers enqueue.

#### GraphQLQueryRunner.run
by default will run with mode `DliteFetchModeConstants.FETCH_MODE_CLIENT`
will run `diffQueries` which is calculated by the `diffRelayQuery` function.
It can also run in REFETCH mode which not even diff the query.

#### diffRelayQuery
diffRelayQuery computes the difference between the data requested in `root` and the data
available in `store`. It returns a minimal set of queries that will fulfill
the difference, or an empty array if the query can be resolved locally.

It Uses `RelayDiffQueryBuilder` to create the diff.

Lets take a look at what happens, based on the TODO example, when you first query with a connection, and then change variables, let's say `first`.

##### First query

{% highlight javascript %}
query TodoList{node(id:"VXNlcjptZQ=="){...F8}}
fragment F0 on Todo{id}
fragment F1 on Todo{complete,id}
fragment F2 on Todo{complete,id,text,...F0,...F1}
fragment F3 on TodoConnection{edges{node{complete,id},cursor},pageInfo{hasNextPage,hasPreviousPage}}
fragment F4 on User{id,totalCount}
fragment F5 on User{id,completedCount}
fragment F6 on User{completedCount,id,totalCount}
fragment F7 on User{id,...F5,...F6}
fragment F8 on User{completedCount,_01:todos(status:"any",first:2){edges{node{id,...F2},cursor},pageInfo{hasNextPage,hasPreviousPage},...F3},totalCount,id,...F4,...F7}
{% endhighlight %}


##### Change first to first + 1

{% highlight javascript %}
query TodoList{node(id:"VXNlcjptZQ=="){...F4}}
fragment F0 on Todo{id}
fragment F1 on Todo{complete,id}
fragment F2 on Todo{complete,id,text,...F0,…F1}
fragment F3 on TodoConnection{edges{node{complete,id},cursor},pageInfo{hasNextPage,hasPreviousPage}}
fragment F4 on User{_00:todos(status:"any",after:"YXJyYXljb25uZWN0aW9uOjA=",first:1){edges{node{id,...F2},cursor},pageInfo{hasNextPage,hasPreviousPage},...F3},id}
{% endhighlight javascript %}

As you can see our second query is much smaller as it's been diffed with what Relay already had in store, it is also doing some magic with the connection. When we incremented the `first` connection arg, relay instead took the global node id of our last todo, set it to the `after` variable, and put our first to `1` instead of requerying the first todos which we already had.

For more information about how Relay actually caches things you can take a look at [Huey Petterson's post][Huey], which goes deeper into it.

#### runQueries
runQueries, as the name says actually run the queries but also initializes callbacks and ready state and also keeps a map of wat has been fetched
{% highlight javascript %}
var remainingFetchMap: {[queryID: string]: PendingFetch} = {};
var remainingRequiredFetchMap: {[queryID: string]: PendingFetch} = {};
{% endhighlight %}

#### storeData.getPendingQueryTracker().add
to send a query, Relay adds a `PendingFetch` object to the PendingQueryTracker, an object to handles a map of `PendingFetch` objects.
{% highlight javascript %}
return new PendingFetch(params, {
  pendingFetchMap: this._pendingFetchMap,
  preloadQueryMap: this._preloadQueryMap,
  storeData: this._storeData
});
{% endhighlight %}


#### PendingFetch
PendingFetch object has a query to resolve. Depending on mode, it will either directly fetch it, or use `this._subtractPending(query);` and then fetch it after.

A `PendingFetch` object subtracts all pending queries from the supplied `query` and returns the resulting difference. The difference can be null if the entire query is pending.

When the pending fetch resolves succesfully, this is called:

{% highlight javascript %}
this._handleSubtractedQuerySuccess.bind(this, subtractedQuery)
{% endhighlight %}

and enqueues a task
{% highlight javascript %}
_this2._storeData.handleQueryPayload(subtractedQuery, response, _this2._forceIndex);
{% endhighlight %}

to update the store and change tracker with the repsonse


#### RelayStoreData.handleQueryPaypload
> Writes the results of a query into the base record store.

{% highlight javascript %}
`writeRelayQueryPayload(writer, query, response);`
{% endhighlight %}

#### writeRelayQueryPayload
Using results from payload: `RelayNodeInterface.getResultsFromPayload`
uses a writter to write every result to the store.

#### RelayQueryWriter.prototype.writePayload
writePayload traverses a query and payload in parallel, writing the results into the store. Using a vistor, updates record Id's using the payload. It updates the change tracker, the store and the queryTracker.

#### runQueries / onResolve

{% highlight javascript %}
if(there are still pending queries)
`setReadyState({done: false, ready: true, stale: false});`
else
`setReadyState({done: true, ready: true, stale: false});`
{% endhighlight %}

I find that how Relay handles diffs and substracts what is already being sent to the server really impressive and cool. Next step is to really dive into how the data is hashed / written to the store, which [Huey's post][huey] explores!

You can find me on Twitter [@\_\_xuorig\_\_][twit] or [Github][xuo]

[twit]: https://twitter.com/__xuorig__
[xuo]: http://github.com/xuorig
[app]: https://github.com/xuorig/my-simple-blogging-app
[huey]: http://hueypetersen.com/posts/2015/09/30/quick-look-at-the-relay-store/
