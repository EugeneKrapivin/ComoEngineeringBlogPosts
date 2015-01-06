Sometimes while using a service bus you'd need to scheduler messages (heartbeats, scheduled maintenance, etc.'). 
Scheduling seems like a simple task, however, done wrongly, might prove to be the greatest nightmare you've ever came across.

There are many different scheduling mechanisms on a variety of service bus implementations. This post will revolve around a [RabbitMQ](http://www.rabbitmq.com/), [Masstransit](http://masstransit-project.com/) and [Quartz.NET](http://www.quartz-scheduler.net/).

**RabbitMQ** is open source message broker software that implements the Advanced Message Queuing Protocol ([AMQP](http://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol)). In simple words, it is a queuing system that transfers messages from one queue to another using a complex system of exchanges and routing rules.[1]

![Service bus illustration](https://cloudcasts4storage.blob.core.windows.net/bookimages/ServiceBusOverview_files/image005.png)
[4]*Message bus illustration*

**MassTransit** is open-source .NET-based Enterprise Service Bus (ESB) software that helps Microsoft developers route messages over service buses such as RabbitMQ, MSMQ, ActiveMQ, etc.[2]

**Quartz.NET** is a pure .NET library written in C# and is a port of very popular open source Java job scheduling framework, Quartz.[3]

Problems and unexpected behavior will occur if more than one service on the bus has the ability to schedule messages for deferred delivery. The reason for this is how RabbitMQ routes and handles its messages.

Suppose we have two services that have the ability to schedule messages, let A and B be those services. Each service is subscribed to the ScheduleMessage and CancelScheduledMessage message. Let C be a service which wants to schedule a message using the ScheduleMessage message, two distinct messages will be posted to the bus, however, since there are two consumers on two different queues, this message will be duplicated for each of those services. Once the time comes to deliver those messages, we encounter our problem - We only scheduled one message, but we are getting two.

![One sender, multiple receivers](http://www.neudesic.com/wp-content/uploads/2014/01/servicebus6.jpg)[5]*One sender, multiple receivers, message fan-out*

If you scale this situation up, you are in a hell-lot-of-trouble:
How do you distinct those messages?
How do you cancel all those messages in time to avoid multiple deliveries of the same message?
How do you handle duplicate messages?
What if the state of a software changes once the first message comes, how do you handle the duplicates?

We'll the solution is quite simple actually - you do not put multiple scheduling services on the same bus.

Building such a service is rather simple using RabbitMQ, Masstransit, Quartz.NET and thier integration packages.

First the service bus should be subscribed to the scheduling messages
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
Once you've initiated the service bus, it's time to use the quartz service to consume those messages and do the heavy lifting of putting the deferred messages on the bus.
```C#
var scheduler = new StdSchedulerFactory().GetScheduler();
scheduler.JobFactory = new MassTransitJobFactory(bus);
scheduler.Start();
```
A full version of the code (using topshelf as the win service bootstrapper) could be found in [this](https://github.com/MassTransit/MassTransit-Quartz/tree/master/src/MassTransit.QuartzService) GitHub repository of Masstransit-Quartz integration.

In conclusion: If the need for scheduling messages arises, you should just put up a dedicated scheduling service for the whole bus and make sure that none of the other services interfere to its functionality by removing their scheduling abilities.

Credits:

* [1] http://en.wikipedia.org/wiki/RabbitMQ
* [2] http://en.wikipedia.org/wiki/MassTransit-Project
* [3] http://www.quartz-scheduler.net/
* [4] http://www.cloudcasts.net
* [5] http://www.neudesic.com

