---
title: "Redis L2 Cache Integration Using Jedis"
date: 2021-06-28T09:34:30-04:00
header:
  teaser: /assets/images/posts/redis_l2_cache/database_cache.jpg
categories:
- blog
tags:
- database
- cache
- redis
- java
- hibernate
---

Redis is fast, easy to manage and quick to deploy. So, let's build a Hibernate L2 cache integration with Redis using the 
popular Java client, Jedis!

![Database Cache](/assets/images/posts/redis_l2_cache/database_cache.jpg)

_<small>Credit: Getty Images</small>_

### What Is L2 Cache?

Hibernate, the popular Java ORM (Object Relational Mapping) framework for Java has L2 cache and it can also use an optional
L2 cache. L1 cache is local to the session, meaning that it is not shared. L2 cache is a shared cache that is turned off by
default. L2 cache has the benefit of potentially being larger than L1 cache since it can be external to the JVM (Java Virtual Machine)
that's using Hibernate.

### Why Build Your Own Hibernate Redis L2 Cache Integration?

Redis is an in-memory datastore that is managed independently of the JVM and it's fast... like really fast. If we are to believe
[Redis's benchmarks](https://redislabs.com/docs/nosql-performance-benchmark/), then Redis is 8x faster than popular NoSQL solutions.
Given that kind of performance, it seems like Redis would be a great candidate to manage L2 cache. 

We wouldn't be alone for thinking that, either. In fact, there is an existing Redis Hibernate L2 cache integration called
Redisson. Redisson has different codecs and it can also be augmented with local caching to boost performance. However, Redisson's
local caching options are limited (without the paid version) and without local caching + a lot of pre-loading, Redisson
[doesn't perform as well as other Java Redis clients, like Jedis](https://www.instaclustr.com/redis-java-clients-and-client-side-caching/).

### Building A Hibernate Redis L2 Cache Integration

As it turns out, it's not really that hard to build a Hibernate L2 cache integration. Basically, we just have to implement
a class that extends Hibernate's RegionFactory. With Hibernate 5, however, that changed slightly. Now, we have to extend
RegionFactoryTemplate. 

Extending RegionFactoryTemplate is how we hook into Hibernate. But, RegionFactoryTemplate also creates a DomainDataStorageAccess
object that defines how the cache is used. So, we will have to both implement DomainDataStorageAccess and extend RegionFactoryTemplate.

_The complete working project discussed below can be found [HERE](https://github.com/BluFlameTech/examples/tree/main/java/cache/jedis-hibernate)._

#### Implementing DomainDataStorageAccess

Our implementation of DomainDataStorageAccess defines how Hibernate interacts with Redis as cache. Unlike Redis, Hibernate
uses Java objects as both keys and values. So, before we can use Redis, we will have to reliably translate objects into Strings
and then back into objects when interacting with Redis. We can do that through serialization as follows.

```java
static String convertObjectToString(Object obj) throws IOException {
    try (ByteArrayOutputStream bos = new ByteArrayOutputStream(); ObjectOutputStream out = new ObjectOutputStream(bos)) {
      out.writeObject(obj);
      out.flush();
      return Base64.getEncoder().encodeToString(bos.toByteArray());
    }
  }

static Object convertStringToObject(String str) throws IOException, ClassNotFoundException {
  try (ByteArrayInputStream bis = new ByteArrayInputStream(Base64.getDecoder().decode(str.getBytes())); ObjectInput in = new ObjectInputStream(bis)) {
    return in.readObject();
  }
}
```

Hmm... where should we put this code? We could just cram it all into our implementation of DomainDataStorageAccess. But,
we're not going to do that. Instead, we will make a cache abstraction that has a Jedis client implementation and that's where
we will do it. It's basically a cache facade with an implementation that decorates Jedis so that we can use it 
with Hibernate's L2 caching. Here's what it looks like.

```java
/**
 * A common/canonical cache interface that supports try w/resources via AutoClosable.
 */
public interface Cache extends AutoCloseable {
  void put(String regionName, Object key, Object value);

  Object get(String regionName, Object key);

  boolean containsObject(String regionName, Object key);

  boolean regionExists(String regionName);

  void purgeRegion(String regionName);

  void purge(String regionName, Object key);
}
```

```java
/**
 * Cache implementation using Jedis client for Redis.
 */
public class JedisCache implements Cache {

  private JedisCacheManager cacheManager;
  Jedis jedis;

  JedisCache(JedisCacheManager cacheManager, Jedis jedis) {
    this.cacheManager = cacheManager;
    this.jedis = jedis;
  }

  @Override
  public void put(String regionName, Object key, Object value) {
    try {
      jedis.hset(regionName, convertObjectToString(key), convertObjectToString(value));
    } catch (IOException exception) {
      throw new CacheException("Unable to serialize object " + key.toString() + " into a String for cache storage");
    }
  }

  @Override
  public Object get(String regionName, Object key) {
    try {
      Optional<String> cachedObject = Optional.ofNullable(jedis.hget(regionName, convertObjectToString(key)));
      if (cachedObject.isPresent()) {
        return convertStringToObject(cachedObject.get());
      }
      return null;
    } catch (IOException | ClassNotFoundException exception) {
      throw new CacheException("Unable to deserialize cached object " + key.toString(), exception);
    }
  }

  @Override
  public boolean regionExists(String regionName) {
    return this.jedis.exists(regionName);
  }

  @Override
  public boolean containsObject(String regionName, Object key) {
    try {
      return this.jedis.hexists(regionName, convertObjectToString(key));
    } catch (IOException exception) {
      throw new CacheException("Unable to eval containsObject; region: " + regionName + " key: " + key, exception);
    }
  }

  @Override
  public void purgeRegion(String regionName) {
    this.jedis.del(regionName);
  }

  @Override
  public void purge(String regionName, Object key) {
    try {
      this.jedis.hdel(regionName, convertObjectToString(key));
    } catch (IOException exception) {
      throw new CacheException("Unable to purge; region: " + regionName + " key: " + key, exception);
    }
  }

  @Override
  public void close() throws Exception {
    this.cacheManager.returnCache(this);
    this.jedis = null;
    this.cacheManager = null;
  }

  static String convertObjectToString(Object obj) throws IOException {
    try (ByteArrayOutputStream bos = new ByteArrayOutputStream(); ObjectOutputStream out = new ObjectOutputStream(bos)) {
      out.writeObject(obj);
      out.flush();
      return Base64.getEncoder().encodeToString(bos.toByteArray());
    }
  }

  static Object convertStringToObject(String str) throws IOException, ClassNotFoundException {
    try (ByteArrayInputStream bis = new ByteArrayInputStream(Base64.getDecoder().decode(str.getBytes())); ObjectInput in = new ObjectInputStream(bis)) {
      return in.readObject();
    }
  }
}
```

You'll notice in the above JedisCache implementation that we're using hset, hget, etc. That's because of Hibernate's
region name namespacing. HSets have a collection name and a key value that it uses to look up values. If we consider
the region name to be a collection name then keys can be used to lookup elements inside of the collection.

You may have also noticed a JedisCacheManager type. This is another abstraction, specially one that can issue access
to the cache... like a pool! From a practical implementation perspective, it is a facade for a Jedis pool.

```java
/**
 * CacheManager implementation for Jedis.
 * This class effectively creates a Pool&lt;Jedis&gt; facade.
 */
public class JedisCacheManager implements CacheManager {

  private static final String CONFIG_PREFIX = "hibernate.cache.redis.";
  private static final String SENTINEL_CONFIG_PREFIX = CONFIG_PREFIX + "sentinel.";
  private static final String STANDALONE_CONFIG_PREFIX = CONFIG_PREFIX + "standalone.";

  Pool<Jedis> jedisPool;

  public JedisCacheManager() {
    this(Map.of());
  }

  /**
   * Constructor creates a Jedis connection pool from config - either standalone or sentinel.
   *
   * @param properties hibernate properties (e.g. from application.yml)
   */
  public JedisCacheManager(Map<String, Object> properties) {
    if (properties.keySet().stream().anyMatch(key -> key.toString().startsWith(SENTINEL_CONFIG_PREFIX))) {
      String master = (String) properties.get(SENTINEL_CONFIG_PREFIX + "master");
      Set<String> sentinels = Set.of(((String) properties.get(SENTINEL_CONFIG_PREFIX + "nodes")).split(",\\s?"));
      String password = (String) properties.get(SENTINEL_CONFIG_PREFIX + "password");
      this.jedisPool = Optional.ofNullable(password)
          .map(pass -> new JedisSentinelPool(master, sentinels, pass))
          .orElse(new JedisSentinelPool(master, sentinels));
      return;
    }

    if (properties.keySet().stream().anyMatch(key -> key.toString().startsWith(STANDALONE_CONFIG_PREFIX))) {
      String host = (String) properties.get(STANDALONE_CONFIG_PREFIX + "host");
      String port = (String) properties.get(STANDALONE_CONFIG_PREFIX + "port");
      this.jedisPool = new JedisPool(host, Integer.parseInt(port));
      return;
    }

    this.jedisPool = new JedisPool();
  }

  @Override
  public Cache getCache() {
    for (int x = 0; x < 3; x++) {
      Jedis jedis = this.jedisPool.getResource();
      if (jedis.isConnected()) {
        return new JedisCache(this, jedis);
      }
    }
    throw new CacheException("Jedis connection pool is unable to get a connection to Redis!");
  }

  @Override
  public void returnCache(Cache cache) {
    if (!(cache instanceof JedisCache)) {
      throw new CacheException("Attempted to return Cache object of type other than JedisCache to JedisCacheManager!");
    }
    JedisCache jedisCache = (JedisCache) cache;
    this.jedisPool.returnResource(jedisCache.jedis);
  }

  @Override
  public boolean isConnected() {
    return !this.jedisPool.isClosed();
  }

  @Override
  public void shutdown() {
    this.jedisPool.close();
  }
}
```

It's the JedisCacheManager that defines where the configuration for Redis comes from. Instead of a separate/special configuration,
we thought it would be nice to have the configuration included with the rest of the Hibernate configuration. In a Spring Boot application.yml
config, it would look something like this.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: com.bluflametech.cache.JedisRegionFactory
          redis:
            sentinel:
              master: mymaster
              nodes: localhost:26379
```

or

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: com.bluflametech.cache.JedisRegionFactory
          redis:
            standalone:
            host: localhost
            port: 6379
```

OK, let's get back to the implementation of DomainDataStorageAccess. With the Cache and the CacheManager defined and implemented
for Jedis, the CacheStorage implementation of DomainDataStorageAccess becomes pretty trivial.

```java
/**
 * Implementation of DomainStorageAccess for CacheManager facilitating interaction with Jedis client.
 */
public class CacheStorage implements DomainDataStorageAccess {

  private final String regionName;
  private final CacheManager cacheManager;

  /**
   * CacheStorage constructor creates CacheStorage instance from regionName and CacheManager.
   *
   * @param regionName   a unique namespace for cached objects
   * @param cacheManager an instance of CacheManager
   */
  public CacheStorage(String regionName, CacheManager cacheManager) {
    this.regionName = regionName;
    this.cacheManager = cacheManager;
    if (!cacheManager.isConnected()) {
      throw new CacheException("CacheManager is not connected to cache!");
    }

    try (Cache cache = cacheManager.getCache()) {
      if (!cache.regionExists(regionName)) {
        cache.put(regionName, "", "");
      }
    } catch (Exception exception) {
      throw new CacheException(exception);
    }
  }

  @Override
  public Object getFromCache(Object key, SharedSessionContractImplementor sharedSessionContractImplementor) {
    try (Cache cache = cacheManager.getCache()) {
      return cache.get(regionName, key);
    } catch (Exception exception) {
      throw new CacheException(exception);
    }
  }

  @Override
  public void putIntoCache(Object key, Object value, SharedSessionContractImplementor sharedSessionContractImplementor) {
    try (Cache cache = cacheManager.getCache()) {
      cache.put(regionName, key, value);
    } catch (Exception exception) {
      throw new CacheException(exception);
    }
  }

  @Override
  public boolean contains(Object key) {
    try (Cache cache = cacheManager.getCache()) {
      return cache.containsObject(regionName, key);
    } catch (Exception exception) {
      throw new CacheException(exception);
    }
  }

  @Override
  public void evictData() {
    try (Cache cache = cacheManager.getCache()) {
      cache.purgeRegion(regionName);
    } catch (Exception exception) {
      throw new CacheException(exception);
    }
  }

  @Override
  public void evictData(Object key) {
    try (Cache cache = cacheManager.getCache()) {
      cache.purge(regionName, key);
    } catch (Exception exception) {
      throw new CacheException(exception);
    }
  }

  @Override
  public void release() {
    evictData();
  }

  String getRegionName() {
    return regionName;
  }

  CacheManager getCacheManager() {
    return cacheManager;
  }
}
```

#### Extending RegionFactoryTemplate

The last thing we need to do is extend RegionFactoryTemplate. That's what let's Hibernate know how to work with our
Jedis-based implementation of their L2 cache. It's also what wires everything else up.

```java
/**
 * Hibernate RegionFactory for Redis L2 cache support via Jedis.
 */
public class JedisRegionFactory extends RegionFactoryTemplate {

  private CacheKeysFactory cacheKeysFactory;
  CacheManager cacheManager;

  /**
   * Sets up Jedis connection pool and initializes RegionFactory.
   *
   * @param settings   hibernate SessionFactoryOptions
   * @param properties hibernate properties (e.g. from spring.jpa.properties in application.yml)
   * @throws CacheException
   */
  @Override
  @SuppressWarnings("unchecked")
  public void prepareForUse(SessionFactoryOptions settings, Map properties) throws CacheException {
    StrategySelector selector = settings.getServiceRegistry().getService(StrategySelector.class);
    cacheKeysFactory = selector.resolveDefaultableStrategy(CacheKeysFactory.class,
        properties.get(Environment.CACHE_KEYS_FACTORY), DefaultCacheKeysFactory.INSTANCE);
    this.cacheManager = new JedisCacheManager((Map<String, Object>) properties);
  }

  @Override
  public boolean isMinimalPutsEnabledByDefault() {
    return true;
  }

  @Override
  protected DomainDataStorageAccess createDomainDataStorageAccess(
      DomainDataRegionConfig regionConfig,
      DomainDataRegionBuildingContext buildingContext) {
    return new CacheStorage(
        qualifyName(regionConfig.getRegionName(), buildingContext.getSessionFactory().getSessionFactoryOptions()),
        cacheManager);
  }

  @Override
  protected StorageAccess createQueryResultsRegionStorageAccess(
      String regionName,
      SessionFactoryImplementor sessionFactory) {
    String defaultedRegionName = defaultRegionName(
        regionName,
        sessionFactory,
        DEFAULT_QUERY_RESULTS_REGION_UNQUALIFIED_NAME,
        LEGACY_QUERY_RESULTS_REGION_UNQUALIFIED_NAMES
    );
    return new CacheStorage(qualifyName(defaultedRegionName, sessionFactory.getSessionFactoryOptions()), cacheManager);
  }

  @Override
  protected StorageAccess createTimestampsRegionStorageAccess(
      String regionName,
      SessionFactoryImplementor sessionFactory) {
    String defaultedRegionName = defaultRegionName(
        regionName,
        sessionFactory,
        DEFAULT_UPDATE_TIMESTAMPS_REGION_UNQUALIFIED_NAME,
        LEGACY_UPDATE_TIMESTAMPS_REGION_UNQUALIFIED_NAMES
    );
    return new CacheStorage(qualifyName(defaultedRegionName, sessionFactory.getSessionFactoryOptions()), cacheManager);
  }

  @Override
  public AccessType getDefaultAccessType() {
    return AccessType.TRANSACTIONAL;
  }

  @Override
  protected void releaseFromUse() {
    cacheManager.shutdown();
  }

  @Override
  public DomainDataRegion buildDomainDataRegion(DomainDataRegionConfig regionConfig, DomainDataRegionBuildingContext buildingContext) {
    verifyStarted();
    return new DomainDataRegionImpl(
        regionConfig,
        this,
        createDomainDataStorageAccess(regionConfig, buildingContext),
        getImplicitCacheKeysFactory(),
        buildingContext);
  }

  @Override
  protected CacheKeysFactory getImplicitCacheKeysFactory() {
    return this.cacheKeysFactory;
  }

  private String qualifyName(String unqualifiedName, SessionFactoryOptions options) {
    assert !RegionNameQualifier.INSTANCE.isQualified(unqualifiedName, options);
    return RegionNameQualifier.INSTANCE.qualify(unqualifiedName, options);
  }

  protected boolean cacheExists(String unqualifiedRegionName, SessionFactoryOptions options) {
    final String qualifiedRegionName = qualifyName(unqualifiedRegionName, options);
    if (this.cacheManager == null) {
      return false;
    }
    try (Cache cache = cacheManager.getCache()) {
      return cache.regionExists(qualifiedRegionName);
    } catch (Exception exception) {
      throw new CacheException(exception);
    }
  }

  //Lovingly poached from JCacheRegionFactory
  //https://github.com/hibernate/hibernate-orm/blob/main/hibernate-jcache/src/main/java/org/hibernate/cache/jcache/internal/JCacheRegionFactory.java
  private String defaultRegionName(String regionName, SessionFactoryImplementor sessionFactory,
                                   String defaultRegionName, List<String> legacyDefaultRegionNames) {
    if (defaultRegionName.equals(regionName)
        && !cacheExists(regionName, sessionFactory.getSessionFactoryOptions())) {
      // Maybe the user configured caches explicitly with legacy names; try them and use the first that exists

      for (String legacyDefaultRegionName : legacyDefaultRegionNames) {
        if (cacheExists(legacyDefaultRegionName, sessionFactory.getSessionFactoryOptions())) {
          SecondLevelCacheLogger.INSTANCE.usingLegacyCacheName(defaultRegionName, legacyDefaultRegionName);
          return legacyDefaultRegionName;
        }
      }
    }
    return regionName;
  }
}
```

The method to pay attention to here is _prepareForUse_. That is what wires everything up. Everything, except for our CacheStorage class
that we implemented earlier. A CacheStorage object is created when a StorageAccess object is requested.
