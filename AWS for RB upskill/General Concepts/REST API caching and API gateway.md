# Caching

- mainly in APPLICATION LAYER
### in-memory caching 
- **Redis**, Memcache

### request level caching
- cache key
- for read heavy API with frequent GET
- expensive queries with complex computation
- paginated data

### conditional caching
- clients receive updated data only when necessary
- HTTP headers 
	- Etag
	- Last modified
### cache invalidation .. redis
- synchronously 
	- when data are updated, cache is updated as well
	- slightly slower writes, write amplification
- asynchronously
	- sync in the background after write to DB is ready
- TTL
	- cache automatically refreshed or removed

| write through | read-heavy workload, where cache accuracy is critical           | slower writes             |
| ------------- | --------------------------------------------------------------- | ------------------------- |
| write behind  | write heavy workloads where cache freshness can tolerate delays | temporary cache staleness |
| TTL based     | data with predictable or time sensitive expiration              | potential for stale data  |

### Ecomerce examaple
- 1st client first loads image - it stores images in cache
- 2nd if browser doesnt have image -> forwarding to CDN 
	- network of servers distributed globaly designed to cache and serve content - images, videos, files 
- 3rd in noone has it, it reaches application server ... cache can 

## Stampedes
- when multiple requests for refreshing invalidated cache under extreme traffic
	- flod of requests could overwhlelm the database
- LOCKING
	- upon cachemiss - acquire a lock for cache key before recomputing the expired page, read from DB, update cache
	- waiting request options:
		- 1 request can wait ti ll updated by another process
		- 2 request can return NOT FOUND and try backup retry
		- 3 system can maintain stale version fo cache item until new
- acquire locking can be challenging
- CACHE REFRESIOG by another process
	- careful for implementation and monitoring
- PROBABILISTIN EARLY EXPIRATION
	- actively trigger expiration before 

## Cache penetration
- request for data non existing in cache nor DB -> results in unnecessary load as system tries to retrieeve it
- implement a PLACEHOLDER for non existing keys
	- set TTL correctly
## Bloom filters

## Cache Crash & Avelanche
- partially or full unavailability of cache - cold start, failure, full crash
- CIRCUIT BREAKER
	- temporarily blocks incomming requests when system is overloaded
- HA CACHE CLUSTER
	- to reduce severity of fullcrash
- CACHE PREWARMING
	- critical afte cold start 
	- proactively poplutae cach before it's put nto service


# API Gateway

## BFF
- dedicated API for each client type
## Proxy
- just pass 

## Aggregator
