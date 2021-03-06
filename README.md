appengine-counter (A Sharded Counter for Google Appengine)
===========================
[![Build Status](https://travis-ci.org/oodlemud/appengine-counter.png)](https://travis-ci.org/oodlemud/appengine-counter)

Appengine-counter is a ShardedCounter implementation for use in Google Appengine.  It offers strongly conistent increment/decrement functionality while maintaining high-throughput via on-the-fly shard configuration.  Appengine-counter uses memcache for fast counter retrieval, all the while being fully backed by the GAE Datastore for incredibly durability and availability.<br/><br/>
Appengine-counter is patterned off of the following <a href="https://developers.google.com/appengine/articles/sharding_counters">article</a> from developer.google.com, but uses Objectify4 for improved maintainability.<br/><br/>
The rationale for a ShardedCounter is as follows (quoted from the linked article):

> When developing an efficient application on Google App Engine, you need to pay attention to how often an entity is updated. While App Engine's datastore scales to support a huge number of entities, it is important to note that you can only expect to update any single entity or entity group about five times a second. That is an estimate and the actual update rate for an entity is dependent on several attributes of the entity, including how many properties it has, how large it is, and how many indexes need updating. While a single entity or entity group has a limit on how quickly it can be updated, App Engine excels at handling many parallel requests distributed across distinct entities, and we can take advantage of this by using sharding."

Thus, when a datastore-backed counter is required (i.e., for counter consistency, redundancy, and availability) we can increment random Counter shards in parallel and achieve a high-throughput counter without sacrificing consistency or availability.  For example, if a particular counter needs to support 100 increments per second, then the application supporting this counter could create the counter with approximately 20 shards, and the throughput could be sustained (this is because any particular entity group in the appengine HRD can support ~5 updates/second).

Features
------
+ <b>Durable</b><br/>
Counter values are stored in the Google Appengine <a href="https://developers.google.com/appengine/docs/python/datastore/structuring_for_strong_consistency">HRD Datastore</a> for data durability and redundancy.  Once an increment or decrement is recorded by the datastore, it's there for good.

+ <b>Available</b><br/>
Since counters are backed by the appengine datastore and appengine itself, counter counts are highly available.

+ <b>Atomic</b><br/>
Counter increment/decrement operations are atomic and will either succeed or fail as a single unit of work.

+ <b>High Performance</b><br/>
Appengine datastore entity groups are limited to ~5 writes/second, so in order to provide a high-throughput counter implementation, a particular counter's 'count' is distributed amongst various counter-shard entities.  Whenever an increment operation is peformed, one of the available shards is chosen to have its count incremented.  In this way, counter throughput is limited only by the number of shards configured for each counter.

+ <b>Smart Caching</b><br/>
Counter values are cached in memcache for high-performance counter reads, and increment/decrement operations update memache so you almost never have to worry about stale counts.

+ <b>Growable Shards --> Higher Throughput</b><br/>
Increasing the number of shards for a particular counter will increase the number of updates/second that the system can handle.  Using appengine-counter, any counter's shard-count can be adjusted in real-time using the appengine datastore viewer.  The more shards, the higher the throughput for any particular counter.

+ <b>Optional Transactionality</b><br/>
By default, counter increment/decrement operations do not happen in an existing datastore transaction.  Instead, a new transaction is always created, which allows the counter to be atomically incremented without having to worry about XG-transaction limits (currently 5 entity groups per Transaction).  However, sometimes it's necessary to increment a counter inside of an XG transaction, and appengine-counter allows for this.

+ <b>Async Counter Deletion<b><br/>
Because some counters may have a large number of counter shards, counter deletion is facilitated in an asynchronous manner using a TaskQueue.  Because of this, counter deletion is eventually consistent, although the counter-status will reflect the fact that it is being deleted, and this information is strongly-consistent.

<b><i><u>Note: The current release of this library is not compatible with Objectify versions prior to version 4.0rc1.  See the changelog for previous version support.</u></i></b>

Getting Started
----------
First, download the latest <a href="https://github.com/oodlemud/appengine-counter/archive/1.0.0.zip">appengine-counter-1.0.0.jar</a> and include it your application's classpath.

Maven users should utilize the following repository and dependency instead:

	<repositories>
		<repository>
			<id>sappenin-objectify-utils</id>
			<url>https://github.com/oodlemud/appengine-counter/tree/master/maven</url>
		</repository>
	</repositories>

    <dependency>
    	<groupId>com.oodlemud.appengine.counter</groupId>
		<artifactId>appengine-counter</artifactId>
		<version>1.0.0</version>
    </dependency>

Sharded counters can be accessed via an implementation of <a href="">ShardedCounterService</a>.  Currently, the only implementation is <a href="">ShardedCounterServiceImpl<a/>, which requires a TaskQueue (the "/default" queue is used by default) if Counter deletion is required.

Queue Configuration
----------
 	<queue-entries>
 		<queue>
			<name>deleteCounterShardQueue</name>
			<!-- add any further queue configuration here -->
		</queue>
	</queue-entries>

Don't forget to add a URL mapping for the default queue, or for the queue mapping you specify below!  By default, the ShardedCounterService uses the default queue URL.  See <a href="https://developers.google.com/appengine/docs/java/taskqueue/overview-push#URL_Endpoints">here</a> for how to configure your push queue URL endpoints.

<i><b>Note that this queue is not required if Counter deletion won't be utilized by your application</b></i>.

Objectify Entity Registration
-----------
Next, be sure to register the appengine-counter entities that are required by the ShardedCounterService, as follows:

	ObjectifyService.factory().register(CounterData.class);
	ObjectifyService.factory().register(CounterShardData.class);

Spring: Default Setup
-------
To utilize the ShardedCounterService with Spring, using the following glue code to provide a default configuration:

	<bean id="shardedCounterService"
		class="com.oodlemud.appengine.counter.service.ShardedCounterServiceImpl">
	</bean>

Spring: Custom Configuration
-------
If you want to control the configuration of the ShardedCounterService, you will need to configure an instance of <b>ShardedCounterServiceConfiguration.Builder</b> as follows:

	<bean id="shardedCounterServiceConfigurationBuilder"
		class="com.oodlemud.appengine.counter.service.ShardedCounterServiceConfiguration.Builder">

		<!-- The number of shards to create when a new counter is created -->
		<property name="numInitialShards">
			<value>3</value>
		</property>

		<!-- The default Memcache expiration for counter objects in milliseconds. -->
		<property name="defaultExpiration">
			<value>300</value>
		</property>

		<!-- The name of the Queue for counter-deletion.  If this property is omitted, the default appengine queue is used -->
		<property name="deleteCounterShardQueueName">
			<value>deleteCounterShardQueue</value>
		</property>

		<!-- The URL callback path that appengine will use to process delete-counter message.  If this property is ommitted, the default appengine queue is used -->
		<property name="relativeUrlPathForDeleteTaskQueue">
			<value>/_ah/queue/deleteCounterShardQueue</value>
		</property>
	</bean>

Next, use the builder defined above to populate a <b>ShardedCounterServiceConfiguration</b>:

	<bean id="shardedCounterServiceConfiguration"
		class="com.oodlemud.appengine.counter.service.ShardedCounterServiceConfiguration">

		<constructor-arg>
			<ref bean="shardedCounterServiceConfigurationBuilder" />
		</constructor-arg>

	</bean>

Finally, use the configuration defined above to create a <b>ShardedCounterService</b> bean.  Notice that you will also need to provide spring-bean configurations for the MemcacheService and CapabilitiesService:

	<bean id="memcacheService" 
		class="com.google.appengine.api.memcache.MemcacheServiceFactory"
		factory-method="getMemcacheService">
	</bean>

	<bean id="capabilitiesService" 
		class="com.google.appengine.api.capabilities.CapabilitiesServiceFactory"
		factory-method="getCapabilitiesService">
	</bean>

	<bean id="shardedCounterService"
		class="com.oodlemud.appengine.counter.service.ShardedCounterService">

		<constructor-arg>
			<ref bean="memcacheService" />
		</constructor-arg>

		<constructor-arg>
			<ref bean="capabilitiesService" />
		</constructor-arg>

		<constructor-arg>
			<ref bean="shardedCounterServiceConfiguration" />
		</constructor-arg>

	</bean>


Guice: Configuration using Annotations
-------
To utilize the a default configuration of the <b>ShardedCounterService</b> with Guice, add the following methods to one of your Guice modules:

	@Provides
	@RequestScoped
	public Ofy provideOfy(OfyFactory fact)
	{
		return fact.begin();
	}

	@Provides
	@RequestScoped
	public MemcacheService provideMemcacheService()
	{
		return MemcacheServiceFactory.getMemcacheService();
	}

	@Provides
	@RequestScoped
	public CapabilitiesService provideCapabilitiesService()
	{
		return CapabilitiesServiceFactory.getCapabilitiesService();
	}

	// The entire app can have a single ShardedCounterServiceConfiguration, though making
	// this request-scoped would allow the config to vary per-request
	@Provides
	@Singleton
	public ShardedCounterServiceConfiguration provideShardedCounterServiceCoonfiguration()
	{
		return new ShardedCounterServiceConfiguration.Builder().withNumInitialShards(2).build();
	}

	// Be safe and make this RequestScoped though not technically needed to be RequestScoped 
	// since ShardedCounterServiceConfiguration and MemcacheService are immutable or utilize 
	// thread-local internally.
	@Provides
	@RequestScoped
	public ShardedCounterService provideShardedCounterService(MemcacheService memcacheService, CapabilitiesService capabilitiesService, ShardedCounterServiceConfiguration config)
	{
		return new ShardedCounterService(memcacheService, capabilitiesService, config);
	}

Don't forget to wire Objectify into Guice:

	public class OfyFactory extends ObjectifyFactory
	{
		/** Register our entity types */
		public OfyFactory()
		{
			final long registrationStartTime = System.currentTimeMillis();
			
			// ///////////////////
			// Register Entities
			// ///////////////////

			// ShardedCounter Entities
			register(Counter.class);
			register(CounterShard.class);

			OfyFactory.logger.info("Objectify Class Registration took "
				+ (System.currentTimeMillis() - registrationStartTime) + " millis");
		}

		@Override
		public Ofy begin()
		{
			return new Ofy(this);
		}
	}

For a more complete example of wiring Guice and Objectify, see the <a href="https://github.com/stickfigure/motomapia">Motomapia Source</a>. 

Guice: Programmatic Configuring without Annotations
-------
To utilize the default configuration of <b>ShardedCounterService</b> with Guice (without using Guice Annotations), add the following classes to your project to create Providers for the ShardedCounterServiceConfiguration, MemcacheService, CapabilitiesService and ShardedCounterService:	

	public class MemcacheServiceProvider implements Provider<MemcacheService>
	{
		@Override
		public MemcacheService get()
		{
			return MemcacheServiceFactory.getMemcacheService();
		}
	}

	public class CapabilitiesServiceProvider implements Provider<CapabilitiesService>
	{
		@Override
		public CapabilitiesService get()
		{
			return CapabilitiesServiceFactory.getCapabilitiesService();
		}
	}

	public class ShardedCounterServiceConfigurationProvider implements
			Provider<ShardedCounterServiceConfiguration>
	{
		@Override
		public ShardedCounterServiceConfiguration get()
		{
			return new ShardedCounterServiceConfiguration.Builder().withNumInitialShards(3).build();
		}
	}

	public class ShardedCounterServiceProvider implements Provider<ShardedCounterService>
	{
		private final ShardedCounterServiceConfiguration config;
		private final MemcacheService memcacheService;
		private final CapabilitiesService capabilitiesService;

		/**
		 * Required-args Constructor.
		 * 
		 * @param config
		 * @param memcacheService
		 * @param capabilitiesService
		 */
		@Inject
		public ShardedCounterServiceProvider(final ShardedCounterServiceConfiguration config,
				final MemcacheService memcacheService, CapabilitiesService capabilitiesService)
		{
			this.config = config;
			this.memcacheService = memcacheService;
			this.capabilitiesService = capabilitiesService;
		}

		@Override
		public ShardedCounterService get()
		{
			return new ShardedCounterService(memcacheService, capabilitiesService, config);
		}
	}

Finally, wire everything together in the configure() method of one of your Guice modules:

	bind(MemcacheService.class).toProvider(MemcacheServiceProvider.class).in(RequestScoped.class);
	bind(CapabilitiesService.class).toProvider(CapabilitiesServiceProvider.class).in(RequestScoped.class);
	// The entire app can have a single ShardedCounterServiceConfiguration, though making
	// this request-scoped would allow the config to vary per-request
	bind(ShardedCounterServiceConfiguration.class).toProvider(ShardedCounterServiceConfigurationProvider.class);
	bind(ShardedCounterService.class).toProvider(ShardedCounterServiceProvider.class).in(RequestScoped.class);


Change Log
----------
**Version 1.0.0**
+ Initial Commit

Authors
-------

**Oodlemud Inc.**
**David Fuelling**

+ http://twitter.com/oodlemud
+ http://github.com/oodlemud


Copyright and License
---------------------

Copyright 2013 Oodlemud Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.