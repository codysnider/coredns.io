+++
title = "cache"
description = "*cache* enables a frontend cache."
weight = 8
tags = ["plugin", "cache"]
categories = ["plugin"]
date = "2022-06-09T08:39:42.8774286"
+++

## Description

With *cache* enabled, all records except zone transfers and metadata records will be cached for up to
3600s. Caching is mostly useful in a scenario when fetching data from the backend (upstream,
database, etc.) is expensive.

*Cache* will change the query to enable DNSSEC (DNSSEC OK; DO) if it passes through the plugin. If
the client didn't request any DNSSEC (records), these are filtered out when replying.

This plugin can only be used once per Server Block.

## Syntax

~~~ txt
cache [TTL] [ZONES...]
~~~

* **TTL** max TTL in seconds. If not specified, the maximum TTL will be used, which is 3600 for
    NOERROR responses and 1800 for denial of existence ones.
    Setting a TTL of 300: `cache 300` would cache records up to 300 seconds.
* **ZONES** zones it should cache for. If empty, the zones from the configuration block are used.

Each element in the cache is cached according to its TTL (with **TTL** as the max).
A cache is divided into 256 shards, each holding up to 39 items by default - for a total size
of 256 * 39 = 9984 items.

If you want more control:

~~~ txt
cache [TTL] [ZONES...] {
    success CAPACITY [TTL] [MINTTL]
    denial CAPACITY [TTL] [MINTTL]
    prefetch AMOUNT [[DURATION] [PERCENTAGE%]]
    serve_stale [DURATION] [REFRESH_MODE]
}
~~~

* **TTL**  and **ZONES** as above.
* `success`, override the settings for caching successful responses. **CAPACITY** indicates the maximum
  number of packets we cache before we start evicting (*randomly*). **TTL** overrides the cache maximum TTL.
  **MINTTL** overrides the cache minimum TTL (default 5), which can be useful to limit queries to the backend.
* `denial`, override the settings for caching denial of existence responses. **CAPACITY** indicates the maximum
  number of packets we cache before we start evicting (LRU). **TTL** overrides the cache maximum TTL.
  **MINTTL** overrides the cache minimum TTL (default 5), which can be useful to limit queries to the backend.
  There is a third category (`error`) but those responses are never cached.
* `prefetch` will prefetch popular items when they are about to be expunged from the cache.
  Popular means **AMOUNT** queries have been seen with no gaps of **DURATION** or more between them.
  **DURATION** defaults to 1m. Prefetching will happen when the TTL drops below **PERCENTAGE**,
  which defaults to `10%`, or latest 1 second before TTL expiration. Values should be in the range `[10%, 90%]`.
  Note the percent sign is mandatory. **PERCENTAGE** is treated as an `int`.
* `serve_stale`, when serve\_stale is set, cache always will serve an expired entry to a client if there is one
  available.  When this happens, cache will attempt to refresh the cache entry after sending the expired cache
  entry to the client. The responses have a TTL of 0. **DURATION** is how far back to consider
  stale responses as fresh. The default duration is 1h. **REFRESH_MODE** controls when the attempt to refresh
  the cache happens. `verified` will first verify that an entry is still unavailable from the source before sending
  the stale response to the client. `immediate` will immediately send the expired response to the client before
  checking to see if the entry is available from the source. **REFRESH_MODE** defaults to `immediate`. Setting this
  value to `verified` can lead to increased latency when serving stale responses, but will prevent stale entries
  from ever being served if an updated response can be retrieved from the source.

## Capacity and Eviction

If **CAPACITY** _is not_ specified, the default cache size is 9984 per cache. The minimum allowed cache size is 1024.
If **CAPACITY** _is_ specified, the actual cache size used will be rounded down to the nearest number divisible by 256 (so all shards are equal in size).

Eviction is done per shard. In effect, when a shard reaches capacity, items are evicted from that shard.
Since shards don't fill up perfectly evenly, evictions will occur before the entire cache reaches full capacity.
Each shard capacity is equal to the total cache size / number of shards (256). Eviction is random, not TTL based.
Entries with 0 TTL will remain in the cache until randomly evicted when the shard reaches capacity.

## Metrics

If monitoring is enabled (via the *prometheus* plugin) then the following metrics are exported:

* `coredns_cache_entries{server, type, zones}` - Total elements in the cache by cache type.
* `coredns_cache_hits_total{server, type, zones}` - Counter of cache hits by cache type.
* `coredns_cache_misses_total{server, zones}` - Counter of cache misses. - Deprecated, derive misses from cache hits/requests counters.
* `coredns_cache_requests_total{server, zones}` - Counter of cache requests.
* `coredns_cache_prefetch_total{server, zones}` - Counter of times the cache has prefetched a cached item.
* `coredns_cache_drops_total{server, zones}` - Counter of responses excluded from the cache due to request/response question name mismatch.
* `coredns_cache_served_stale_total{server, zones}` - Counter of requests served from stale cache entries.
* `coredns_cache_evictions_total{server, type, zones}` - Counter of cache evictions.

Cache types are either "denial" or "success". `Server` is the server handling the request, see the
prometheus plugin for documentation.

## Examples

Enable caching for all zones, but cap everything to a TTL of 10 seconds:

~~~ corefile
. {
    cache 10
    whoami
}
~~~

Proxy to Google Public DNS and only cache responses for example.org (or below).

~~~ corefile
. {
    forward . 8.8.8.8:53
    cache example.org
}
~~~

Enable caching for `example.org`, keep a positive cache size of 5000 and a negative cache size of 2500:

~~~ corefile
example.org {
    cache {
        success 5000
        denial 2500
    }
}
~~~
