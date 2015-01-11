When you're using a service bus, scheduling messages such as heartbeats and scheduled maintenance may seem like a simple task. But if you run into the problem we'll talk about here, it can turn into your greatest nightmare.

There are many different scheduling mechanisms on a variety of service bus implementations. This post will focus on [RabbitMQ](http://www.rabbitmq.com/), [Masstransit](http://masstransit-project.com/) and [Quartz.NET](http://www.quartz-scheduler.net/).

**RabbitMQ** is open source message broker software that implements the Advanced Message Queuing Protocol ([AMQP](http://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol)). Put simply, it is a queuing system that transfers messages from one queue to another using a complex system of exchanges and routing rules.[1]

![Service bus illustration](https://cloudcasts4storage.blob.core.windows.net/bookimages/ServiceBusOverview_files/image005.png)

[4]*Message bus illustration*

**MassTransit** is open source .NET-based Enterprise Service Bus (ESB) software that helps Microsoft developers route messages over service buses such as RabbitMQ, MSMQ, ActiveMQ, etc.[2]

**Quartz.NET** is a pure .NET library written in C# and is a port for Quartz, the very popular open source Java job scheduling framework.[3]

The problem shows up when more than one service on the bus can schedule messages for deferred delivery. This is due to the way that RabbitMQ routes and handles its messages.

Suppose we have two services that can schedule messages,  A and B. Each service is subscribed to the ScheduleMessage and CancelScheduledMessage message. Let's say that another service, C, wants to schedule a message using the ScheduleMessage message. Two distinct messages will be posted to the bus, however, since there are two consumers on two different queues, this message will be duplicated for each of those services. When the time comes to deliver those messages, we see the problem: We only scheduled one message, but we get two.

![One sender, multiple receivers](http://www.neudesic.com/wp-content/uploads/2014/01/servicebus6.jpg)[5]*One sender, multiple receivers, message fan-out*

If you scale this situation up, you could be in a lot of trouble:
How do you distinguish between the messages?
How do you cancel all the messages in time to avoid multiple deliveries of the same message?
How do you handle duplicate messages?
What if the state of the software changes once the first message comes. How do you handle the duplicates?

Actually, the solution is quite simple. You don't put multiple scheduling services on the same bus.

Building a service like this is rather simple using RabbitMQ, Masstransit, Quartz.NET, and their integration packages.

First, the service bus should be subscribed to the scheduling messages:
```C#
bus = ServiceBusFactory.New(x =>
{
	x.UseRabbitMq();
    x.ReceiveFrom(_controlQueueUri); // your queue
    x.Subscribe(s =>
	{
		/* other message subscriptions */
		s.Consumer(() => new ScheduleMessageConsumer(_scheduler));
		s.Consumer(() => new CancelScheduledMessageConsumer(_scheduler));
		/* other message subscriptions */
	});
}
```
Once you've initiated the service bus, it's time to use the Quartz service to consume those messages and do the heavy lifting, meaning putting the deferred messages on the bus.
```C#
var scheduler = new StdSchedulerFactory().GetScheduler();
scheduler.JobFactory = new MassTransitJobFactory(bus);
scheduler.Start();
```
A full version of the code (using Topshelf as the Windows service bootstrapper) can be found in [this](https://github.com/MassTransit/MassTransit-Quartz/tree/master/src/MassTransit.QuartzService) GitHub repository of Masstransit-Quartz integration.

In conclusion: If you need to schedule messages, we recommend putting up a dedicated scheduling service for the whole bus and making sure that none of the other services interfere with its functionality by removing their scheduling abilities.

Credits:

* [1] http://en.wikipedia.org/wiki/RabbitMQ
* [2] http://en.wikipedia.org/wiki/MassTransit-Project
* [3] http://www.quartz-scheduler.net/
* [4] http://www.cloudcasts.net
* [5] http://www.neudesic.com

