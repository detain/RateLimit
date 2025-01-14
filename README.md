# rate-limit

[![GitHub Build](https://github.com/detain/rate-limit/actions/workflows/php.yml/badge.svg)](https://github.com/detain/rate-limit/actions/workflows/php.yml)
[![Travis Build](https://app.travis-ci.com/detain/rate-limit.svg?branch=master)](https://app.travis-ci.com/detain/rate-limit)
[![Coverage Status](https://coveralls.io/repos/github/detain/rate-limit/badge.svg?branch=master)](https://coveralls.io/github/detain/rate-limit?branch=master)
[![Code Climate](https://codeclimate.com/github/detain/rate-limit/badges/gpa.svg)](https://codeclimate.com/github/detain/rate-limit)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/detain/rate-limit/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/detain/rate-limit/?branch=master)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/659523f63e16487ea71f6b763908d09e)](https://www.codacy.com/app/detain/rate-limit)

[![Latest Stable Version](https://poser.pugx.org/detain/rate-limit/version)](https://packagist.org/packages/detain/rate-limit)
[![Total Downloads](https://poser.pugx.org/detain/rate-limit/downloads)](https://packagist.org/packages/detain/rate-limit)
[![Latest Unstable Version](https://poser.pugx.org/detain/rate-limit/v/unstable)](//packagist.org/packages/detain/rate-limit)
[![License](https://poser.pugx.org/detain/rate-limit/license)](https://packagist.org/packages/detain/rate-limit)
[![Monthly Downloads](https://poser.pugx.org/detain/rate-limit/d/monthly)](https://packagist.org/packages/detain/rate-limit)
[![Daily Downloads](https://poser.pugx.org/detain/rate-limit/d/daily)](https://packagist.org/packages/detain/rate-limit)

PHP Rate Limiting library with both Token Bucket and Leaky Bucket Algorithms, minimal external dependencies, and many storage backends.

# Algorithms

- [x] [Token Bucket Algorithm](https://en.wikipedia.org/wiki/Token_bucket) Token Bucket is an algorithm which works as follows:
  - There is a bucket.
  - A token is added to the bucket every 1/r seconds.
  - The bucket can hold at the most b tokens. If a token arrives when the bucket is full, it is discarded.
  - When a packet (network layer PDU) of n bytes arrives,
	- If at least n tokens are in the bucket, n tokens are removed from the bucket, and the packet is sent to the network.
	- if fewer than n tokens are available, no tokens are removed from the bucket, and the packet is considered to be non-conformant.
	
- [x] [Leaky Bucket Algorithm](https://en.wikipedia.org/wiki/Leaky_bucket) Leaky Bucket is an algorithm which works as follows:
  - There is a bucket.
  - The bucket has a defined leak and defined capacity.
  - The bucket leaks at a constant rate.
  - Overflows when full, will not add other drops to the bucket.
  

# Storage Adapters

The RateLimiter needs to know where to get/set data. 

Depending on which adapter you install, you may need to install additional libraries (predis/predis or tedivm/stash) or PHP extensions (e.g. Redis, Memcache, APCu)


- [APCu](https://pecl.php.net/package/APCu)
- [Redis](https://pecl.php.net/package/redis) or [Predis](https://github.com/nrk/predis)
- [Stash](http://www.stashphp.com) (This supports many drivers - see http://www.stashphp.com/Drivers.html )
- [Memcached](http://php.net/manual/en/intro.memcached.php)


# Installing via Composer

````shell
curl -sS https://getcomposer.org/installer | php
composer.phar require detain/rate-limit
````

# Usage

## Token Bucket
````php
require 'vendor/autoload.php';

use \Detain\RateLimit\RateLimit;
use \Detain\RateLimit\Adapter\APCu as APCAdapter;
use \Detain\RateLimit\Adapter\Redis as RedisAdapter;
use \Detain\RateLimit\Adapter\Predis as PredisAdapter;
use \Detain\RateLimit\Adapter\Memcached as MemcachedAdapter;
use \Detain\RateLimit\Adapter\Stash as StashAdapter;


$adapter = new APCAdapter(); // Use APC as Storage
// Alternatives:
//
// $adapter = new RedisAdapter((new \Redis()->connect('localhost'))); // Use Redis as Storage
//
// $adapter = new PredisAdapter(new \Predis\Predis(['tcp://127.0.0.1:6379'])); // Use Predis as Storage
//
// $memcache = new \Memcached();
// $memcache->addServer('localhost', 11211);
// $adapter = new MemcacheAdapter($memcache); 
//
// $stash = new \Stash\Pool(new \Stash\Driver\FileSystem());
// $adapter = new StashAdapter($stash);

$rateLimit = new RateLimit("myratelimit", 100, 3600, $adapter); // 100 Requests / Hour

$id = $_SERVER['REMOTE_ADDR']; // Use client IP as identity
if ($rateLimit->check($id)) {
  echo "passed";
} else {
  echo "rate limit exceeded";
}
````

## Leaky Bucket

### Basic usage
``` php
<?php

use \Detain\RateLimit\LeakyBucket;
use \Detain\RateLimit\Adapter\Redis as RedisAdapter;

$adapter = new RedisAdapter((new \Redis()->connect('localhost'))); // Use Redis as Storage

// Define the bucket
$settings = [
	'capacity' => 10,
	'leak'     => 1
];

// Create the bucket
$bucket = new LeakyBucket('example-bucket', $storage, $settings);

// Fill the bucket
$bucket->fill();

// Check if it's full
if ($bucket->isFull()) {
	header('HTTP/1.1 429 Too Many Requests');
	exit '<!doctype html><html><body><h1>429 Too Many Requests</h1><p>You seem to be doing a lot of requests. You\'re now cooling down.</p></body></html>';
}

// ...
```

### Other functionality
You can also do great stuff with it through the methods that LeakyBucket provides.

``` php
// Get capacity information
$capacityTotal = $bucket->getCapacity();
$capacityLeft  = $bucket->getCapacityLeft();
$capacityUsed  = $bucket->getCapacityUsed();

// Get the drops/second that the bucket leaks
$leakPerSecond = $bucket->getLeak();

// Get the last timestamp from when the bucket was updated
$timestamp = $bucket->getLastTimestamp();

// Set additional data
$bucket->setData(
	[
		'timeout' => 3600
	]
);

// Get additional data
$data = $bucket->getData();

// Update the bucket with the leaked drops
$bucket->leak();

// Remove excess drops
$bucket->overflow();

// Update the bucket's timestamp manually
$bucket->touch();

// Fill the bucket with one drop
$bucket->fill();

// Fill the bucket with 5 drops
$bucket->fill(5);

// Spill one drop from the bucket
$bucket->spill();

// Spill 5 drops from the bucket
$bucket->spill(5);

// Remove the bucket's content
$bucket->reset();

// Force save
$bucket->save();
```

# References

## Continuation or Expansion of these Projects

- [this code is forked from DavidGoodwin/RateLimit](https://github.com/DavidGoodwin/RateLimit)
- [which was built on touhonoob/RateLimit](https://github.com/touhonoob/RateLimit)
- [and then i merged it with jeroenvisser101/LeakyBucket](https://github.com/jeroenvisser101/LeakyBucket)

## Knowledge

- [Token Bucket Algorithm](http://en.wikipedia.org/wiki/Token_bucket)
- [Leaky Bucket Algorithm](https://en.wikipedia.org/wiki/Leaky_bucket)
- [stackoverflow post about Rate Limiting](http://stackoverflow.com/a/668327/670662)

## PSR-7 Middleware Handlers to Base the Future PSR7 Support Off Of

- [nikolaposa/rate-limit](https://github.com/nikolaposa/rate-limit) 🚔 Standalone component that facilitates rate-limiting functionality. Also provides a middleware designed for API and/or other application endpoints. 
- [robwittman/leaky-bucket-rate-limiter](https://github.com/robwittman/leaky-bucket-rate-limiter) PSR-7 Leaky Bucket Rate Limiter - This middleware enables API Rate-Limiting based on a Leaky Bucket algorithm.
