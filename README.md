# ed-laravel-dragonfly
Using Laravel with Dragonfly

## Introduction

Dragonfly is a drop-in Redis replacement that delivers far better performance with far fewer servers. A single node can handle millions of queries per second and up to 1TB of in-memory data. In this blog post, we will explore how to use Dragonfly with Laravel, one of the most widely used and well-known web frameworks.

Dragonfly maintains full compatibility with the Redis interface, meaning Laravel developers can integrate it as a cache and queue driver without a single line of code change. This seamless integration offers an effortless upgrade path with substantial benefits.

So, whether you are a seasoned Laravel veteran or just starting out, join us as we step into the world of Dragonfly and Laravel.


## Getting Started

Let’s start by setting up a new Dragonfly instance. Visit our documentation [here](https://www.dragonflydb.io/docs/getting-started) to download an image or the binary and have a Dragonfly instance up and running in no time. Once the Dragonfly instance is operational and reachable, integrating it with your Laravel project is a breeze. Luckily, Laravel already has full support for Redis, so all of its drivers can be reused. To use Dragonfly in your Laravel application, start by updating the .env file with the following configurations.

For caching and session management:

```
CACHE_DRIVER=redis
SESSION_DRIVER=redis
```

To integrate Dragonfly as the queue driver as well:

```
QUEUE_CONNECTION=redis
```

Even though we are using redis as the driver value, Dragonfly is designed to be a direct replacement for Redis, so no additional driver installation is required. With the driver set, the next step is to ensure Laravel can communicate with the Dragonfly instance. This involves updating the .env file again with the correct connection details:

- REDIS_HOST: The hostname or IP address of the Dragonfly server.
- REDIS_PORT: The port on which the Dragonfly instance is running.
- REDIS_PASSWORD: The password for the Dragonfly instance, if set.

Here’s an example configuration:

```
REDIS_HOST=127.0.0.1 # Replace with Dragonfly host
REDIS_PORT=6379      # Replace with Dragonfly port
REDIS_PASSWORD=null  # Replace with Dragonfly password if applicable
```

After updating these settings, verify the connection by running a simple operation like INFO in Laravel. If you encounter any connectivity issues, double-check the host, port, and password values. Also, ensure that the Dragonfly server is running and accessible from your Laravel application's environment.

```
use Illuminate\Support\Facades\Redis;

// Run the INFO command and print the Dragonfly version.
Redis::info()["dragonfly_version"];
```

## Higher Efficiency as a Cache

Caching commonly accessed values is one of the primary uses of in-memory databases like Dragonfly and Redis due to their fast response times. This is where Dragonfly shines, especially in scenarios involving a large number of keys and clients, typical as a central cache of multi-node systems or microservices.

Dragonfly’s prowess is not just in its speed but also in its intelligent memory management. A standout feature is the cache mode, designed specifically for scenarios where maintaining a lean memory footprint is as crucial as performance. In this mode, Dragonfly smartly evicts the least recently accessed values when it detects low memory availability, ensuring efficient memory usage without sacrificing speed. You can read more about the eviction algorithm in the [Dragonfly Cache Design blog post](https://www.dragonflydb.io/blog/dragonfly-cache-design).

Activating the cache mode is straightforward. Here are the flags you would use to run Dragonfly in this mode, with a memory cap of 12GB:

```
./dragonfly --cache_mode --maxmemory=12G
```

Consider a scenario where your application needs to handle a high volume of requests with a vast dataset. In such cases, the Dragonfly cache mode can efficiently manage memory usage while providing rapid access to data, ensuring your application remains responsive and agile.

API-wise, all functionality of the Laravel Cache facade should be supported. For example, to store a given key and value with a specific expiration time, the following snippet can be used:

```
use Illuminate\Support\Facades\Cache;

// Store a value with a 10 minute expiration time.
Cache::put("key", "value", 600);
```

## Memory Usage

One of the benefits of using Dragonfly as a cache is its measurably lower memory usage for most use cases. Let’s conduct a simple experiment and fill both Redis and Dragonfly with random strings, measuring their total memory usage after filling them with data.

 | Dataset | Dragonfly | Redis3 | 
 |---------|-----------|--------|
 | Million Values of Length | 10002.75GB | 3.17GB15 | 
 | Million Values of Length | 2003.8GB | 4.6GB |

After conducting the experiment, we’ve observed that Dragonfly’s memory usage is up to 20% lower compared to Redis under similar conditions. This allows you to store significantly more useful data with the same memory requirements, making the cache more efficient and achieving higher coverage. You can read more about Dragonfly throughput benchmarks and memory usage in the [Redis vs. Dragonfly Scalability and Performance blog post](https://www.dragonflydb.io/blog/scaling-performance-redis-vs-dragonfly).

## Snapshotting

Beyond just lower memory usage, Dragonfly also demonstrates remarkable stability during snapshotting processes. Snapshotting, particularly in busy instances, can be a challenge in terms of memory management. With Redis, capturing a snapshot on a highly active instance might lead to increased memory usage. This happens because Redis needs to copy memory pages, even those that have only been partially overwritten, resulting in a spike in memory usage.

Dragonfly, in contrast, takes a more adaptive approach to snapshotting. It intelligently adjusts the order of snapshotting based on incoming requests, effectively preventing any unexpected surges in memory usage. This means that even during intensive operations like snapshotting, Dragonfly maintains a stable memory footprint, ensuring consistent performance without the risk of sudden memory spikes. You can read more about the Dragonfly snapshotting algorithm in the Balanced vs. Unbalanced blog post.

## Key Stickiness

Dragonfly also introduces a new feature with its custom STICK command. This command is particularly useful in instances running in cache mode. It enables specific keys to be marked as non-evicting, irrespective of their access frequency.

This functionality is especially handy for storing seldom-accessed yet important data. For example, you can reliably keep auxiliary information, like dynamic configuration values, directly on your Dragonfly instance. This eliminates the need for a separate datastore for infrequently used but crucial data, streamlining your data management process.

```
// Storing a value in the Dragonfly instance with stickiness.
Redis::transaction(function (Redis $redis) {
    $redis->set('server-dynamic-configuration-key', '...');
    $redis->command('STICK', 'server-dynamic-configuration-key');
});

// ...// Will always return a value since the key cannot be evicted.
$redis->get('server-dynamic-configuration-key');
```

## Enhanced Throughput in Queue Management

Dragonfly, much like Redis, is adept at managing queues and jobs. As you might have already guessed, the transition to using Dragonfly for this purpose is seamless, requiring no code modifications. Consider the following example in Laravel, where a podcast processing job is dispatched:

```
use App\Jobs\ProcessPodcast;

$podcast = Podcast::create(/* ... */);
ProcessPodcast::dispatchSync($podcast);
```

Both Dragonfly and Redis are capable of handling tens of thousands of jobs per second with ease.

For those aiming to maximize performance, it’s important to note that using a single job queue won’t yield significant performance gains. To truly leverage Dragonfly’s capabilities, multiple queues should be utilized. This approach distributes the load across multiple Dragonfly threads, enhancing overall throughput.

However, a common challenge arises when keys from the same queue end up on different threads, leading to increased latency. To counter this, Dragonfly offers the use of hashtags in queue names. These hashtags ensure that jobs in the same queue (which uses the same hashtag) are automatically assigned to specific threads, much like in a Redis Cluster environment, thereby reducing latency and optimizing performance. To learn more about hashtags, check out the [Running BullMQ with Dragonfly blog post](https://www.dragonflydb.io/blog/running-bullmq-with-dragonfly-part-1-announcement), which has a detailed explanation of hashtags and their benefits, while Dragonfly is used as a backing store for message queue systems.

As a quick example, to optimize your queue management with Dragonfly, start by launching Dragonfly with specific flags that enable hashtag-based locking and emulated cluster mode:

```
./dragonfly --lock_on_hashtags --cluster_mode=emulated
```

Once Dragonfly is running with these settings, incorporate hashtags into your queue names in Laravel. Here’s an example:

```
ProcessPodcast::dispatch($podcast)->onQueue('{podcast_queue}');
```

By using hashtags in queue names, you ensure that all messages belonging to the same queue are processed by the same thread in Dragonfly. This approach not only keeps related messages together, enhancing efficiency, but also allows Dragonfly to maximize throughput by distributing different queues across multiple threads.

This method is particularly effective for systems that rely on Dragonfly as a message queue backing store, as it leverages Dragonfly’s multi-threaded architecture to handle a higher volume of messages more efficiently.
