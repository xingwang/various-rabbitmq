# various-rabbitmq
---

Some RabbitMQ tidbits for quick referencing.

## setup

MRI Ruby using [bunny](http://rubybunny.info/articles/guides.html)

```ruby
require 'bunny'
conn = Bunny.new('amqp://localhost')
conn.start
ch = conn.create_channel
some_exchange = ch.topic('some_exchange_name', :durable => true)
some_queue = ch.queue('some_queue_name', :durable => true).bind(some_exchange, :routing_key => '#')
some_queue.subscribe(:block => true, :manual_ack => true) do |delivery_info, properties, msg|
	#consumption logic
end
```

JRuby using [march_hare](http://rubymarchhare.info/articles/guides.html)

```ruby
require 'march_hare'
conn = MarchHare.connect(:uri =>'amqp://localhost')
ch = conn.create_channel
some_exchange = ch.topic('some_exchange_name', :auto_delete => false)
some_queue = ch.queue('some_exchange_name', :auto_delete => false).bind(some_exchange, :routing_key => '#')
some_queue.subscribe(:manual_ack => true) do |metadata, payload|
	#consumption logic
end
```

Node.js using [amqplib](http://www.squaremobius.net/amqp.node/channel_api.html) callback api

```javascript
const amqp = require('amqplib/callback_api');

amqp.connect('amqp://localhost', (err, conn) => {
  conn.createConfirmChannel((err, ch) => {
    ch.assertExchange('someExchangeName', 'topic', { durable: false});
    ch.assertQueue('someQueueName', {durable: false}, (err, q) => {
      ch.bindQueue(q.queue, 'someExchangeName', '#');
      ch.consume(q.queue, (msg) => {
        //consumption logic
      });
    });
  });
});
```

## message properties

Bunny subscribe returns 3 parameters: [delivery_info](http://rubybunny.info/articles/queues.html#accessing_message_delivery_information), [metadata](http://rubybunny.info/articles/queues.html#accessing_message_properties_metadata), and the payload.

March Hare subscribe returns 2 parameters: [metadata](http://rubymarchhare.info/articles/queues.html#accessing_message_delivery_information) and the payload.

Amqplib consume returns just 1 message object with all info on it.

```javascript
{
  "fields": {
    "consumerTag": "amq.ctag-T2Tz6NF9_7vheeiuGcAsCg",
    "deliveryTag": 1,
    "redelivered": false,
    "exchange": "someExchangeName",
    "routingKey": "someRoutingKey"
  },
  "properties": {
    "headers": {
      "x-death": [
      		//dead letter header info as object sorted by activity
      ]
    },
    "expiration": "someExpirationInMs"
  },
  "content": {
    "type": "Buffer",
    "data": [string buffer]
  }
}
```

## differences

All 3 libraries can do everything you need from RabbitMQ and all the standard extensions. However, if you want some extra nice-to-haves, the ruby/jruby libraries have a couple more things.

One of the most important is support for auto recovery when there's a connection problem. For amqplib, you will need to roll your own using the on close event followed by more custom code for any exchange/queue recreations. With the ruby versions, you get all that for free. In fact, even the Java and .Net implementations have this feature as well although they are opt-in as opposed to Ruby verions' opt-out policy.

Another interesting thing the ruby versions have is support for the custom exchange types, consistent-hash and x-random. You can read more about them [here](http://rubybunny.info/articles/exchanges.html#custom_exchange_types).

For JRuby's March Hare, not only do you get the usual push API in the queue subscribe, you also have the ability to do a pull using the queue pop.

Amqplib does have a promise API but the return value is not always what you expect for certain operations. Also, due to the nature of promises, it's better to just stick with callbacks so you're covered on all types of scenarios especially since you can't mix the 2 APIs.

As far as documentation goes, amqplib is better organized for navigating through the API faster. It is also more amusing to read. However, for concepts, either of the Ruby versions are better reads.

## other stuff

Once you get beyond a certain amount of queues, your topology design might come into play. A few great reads on performance and scalability may be found [here](http://spring.io/blog/2011/04/01/routing-topologies-for-performance-and-scalability-with-rabbitmq/), [here](https://www.rabbitmq.com/blog/2010/10/19/exchange-to-exchange-bindings/), and [here](http://skillachie.com/2014/06/27/rabbitmq-exchange-to-exchange-bindings-ampq/). The basic gist is instead of doing a ton of queues with a ton of bindings, do more exchange to exchange bindings where possible. This promotes decoupling, reduce binding churn, and increase performance. However, you can pretty much disregard all that if your app is always going to be on the small scale.
