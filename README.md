## TrendrrNSQCLient
=============

- Please Note, I am not actively maintaining this codebase.  If anyone wants to maintain a fork I can link to it.

A fast netty based java client for [nsq][nsq].  We developed this client to use in various places in the trendrr.tv and curatorr.com stacks.
It is currently deployed in production.  It produces and consumes billions of messages per day. 


## Consumer

Example usage:

```
NSQLookup lookup = new NSQLookupDynMapImpl();
lookup.addAddr("localhost", 4161);
NSQConsumer consumer = new NSQConsumer(lookup, "speedtest", "dustin", new NSQMessageCallback() {
            
    @Override
    public void message(NSQMessage message) {
        System.out.println("received: " + message);            
        //now mark the message as finished.
        message.finished();
        
        //or you could requeue it, which indicates a failure and puts it back on the queue.
        //message.requeue();
    }           
    @Override
    public void error(Exception x) {
        //handle errors
        log.warn("Caught", x);
    }
});
        
consumer.start();
```


## Producer

Example usage: 

```
NSQProducer producer = new NSQProducer().addAddress("localhost", 4150, 1);            
producer.start();
for (int i=0; i < 50000; i++) {
    producer.produce("speedtest", ("this is a message" + i).getBytes());
}
```

The producer also has a Batch collector that will collect messages until some threshold is reached (currently maxbytes or maxmessages) then send as a MPUB request.  This gives much greater throughput then producing messages one at a time.

```
producer.configureBatch("speedtest", 
                new BatchCallback() {
                    @Override
                    public void batchSuccess(String topic, int num) {
                    }
                    @Override
                    public void batchError(Exception ex, String topic, List<byte[]> messages) {
                        ex.printStackTrace();   
                    }
                }, 
            batchsize, 
            null, //use default maxbytes 
            null //use default max seconds
        );

producer.start();
for (int i=0; i < iterations; i++) {
    producer.produceBatch("speedtest", ("this is a message" + i).getBytes());
}
```

## Connection parameters

* `connection.messagesPerBatch` is the `RDY` value that is maintained for each nsqd connection.  Note that when throttling messages, nsqd subtracts the number of messages that have not yet been ack'd from this value.  `messagesPerBatch` can be used to limit the number of threads that will be used simultaneously for message callbacks (the maximum is messagesPerBatch * connections).

## Threading

IO and protocol handling are done in threads managed by netty.

Producers may write to an NSQClient from any thread (according to the netty thread model).

Consumer callbacks are run in a thread provided by a configurable Executor, which is by default a CachedThreadPool.  Use caution configuring the executor - if the number of concurrent threads is less than (messagesPerBatch * connections), tasks may back up in the executor's work queue.  Backing up messsages in a consumer isn't ideal - there may be other consumers ready to process them.

## Clean shutdown

The consumer has a `shutdown` method which attempts a clean shutdown: no new messages will be consumed, but message callbacks that are already in process will be allowed to complete.  To perform a clean shutdown on SIGINT, add a shutdown hook like the following to your main class:

```
		Runtime.getRuntime().addShutdownHook(new Thread() {
			@Override
			public void run() {
				consumer.shutdown();

				try {
					if (consumer.awaitTermination(10, TimeUnit.SECONDS)) {
						Runtime.getRuntime().halt(0);
					}
				} catch (InterruptedException e) {
					log.warn("Timeout waiting for consumer to shutdown");
				}
			}
		});
```

## TODO

* Backoff
* Allow limiting the total number of messages processed concurrently across connections (total_rdy_count)
* Support total_rdy_count < connections which will allow thread usage to be capped even if the number of nsqd producers grows.
