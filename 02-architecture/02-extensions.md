# Extensions

This document describes how to write extensions for Squidex. We use interfaces for all components that could be replaced and register them in the service locator.

We use the dependency injection (see https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) combined with Autofac.

Our recommendation is to create a custom Autofac Module for your extensions:

    public class MyModule : Module
    {
        private IConfiguration Configuration { get; }

        public StoreModule(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        protected override void Load(ContainerBuilder builder)
        {
            builder.RegisterType<SqlEventStore>()
                .As<IEventStore>()
                .As<IExternalSystem>()
                .SingleInstance();
        }
    }

## Infrastructure Extensions

### IAssetStore

The `Squidex.Infrastructure.Assets.IAssetStore` interface is used to encapsulate storage solutions for assets. Currently there are the following implementations:

* `AzureBlobAssetStore`: Store the assets in azure blob storage.
    * Read more: https://azure.microsoft.com/en-us/services/storage/blobs/
* `GoogleCloudAssetStore`: Store the assets in Google cloud.
    * Read more: https://cloud.google.com/storage/
* `FolderAssetStore`: Store the assets in the file system.

Recommended implementations:

* Amazon S3

### IPubSub

The `Squidex.Infrastructure.IPubSub` interface is used to define a very simple mechanism for distributing events between nodes in a multi-node deployment. For example there is the event store notifies the event handler that there is an update. There is a also a local cache on each node and we use the PubSub mechanism to keep them in sync. Currently there are the following implementations:

* `Squidex.Infrastructure.RedisPubSub`: Publishs messages with Redis. 
    * Read more: https://redis.io/
* `Squidex.Infrastructure.InMemoryPubSub`: Publishs messages with an in-memory-queue. Can only be used for single-node deployments.

Recommended implementations:

* Amazon SNS
* Azure Queue
* Azure Servicebus

### IEventStore

The `Squidex.Infrastructure.CQRS.Events.IEventStore` is our abstraction for different event store implementations. You can append to events, query them or subscribe to events. Dependending on your implementation you might want to use the pub-sub system for subscriptions. The notification mechanism is provided by the `Squidex.Infrastructure.CQRS.Events.IEventNotifier` interface. Currently there are the following implementations:

* `Squidex.Infrastructure.CQRS.Events.MongoEventStore`: Implementation for MongoDb. 
    * Read more: https://docs.mongodb.com/ecosystem/drivers/csharp/
* `Squidex.Infrastructure.CQRS.Events.GetEventStore`: Implementation for EventStore. 
    * Read more: https://geteventstore.com/

Recommended implementations:

* SQL Databases

The `Squidex.Infrastructure.CQRS.Events.IEventConsumerInfoRepository` defines the contract for a system to store the current state of each event consumer.

### IAssetThumbnailGenerator

The `Squidex.Infrastructure.Assets.IAssetThumbnailGenerator` interface encapsulates image transformations. We only have an implementation for ImageSharp: https://github.com/SixLabors/ImageSharp

## Repositories

You can provide other implementations for repositories, e.g. for Elastic Search or SQL:

* `Squidex.Domain.Apps.Read.Apps.Repositories.IAppRepository`: Stores all app information, including settings like languages, contributors and clients.
* `Squidex.Domain.Apps.Read.Assets.Repositories.IAssetRepository`: Stores all information about assets, but not the content itself.
* `Squidex.Domain.Apps.Read.Assets.Repositories.IAssetStatsRepository`: Stores usage statistics about assets.
* `Squidex.Domain.Apps.Read.Contents.Repositories.IContentRepository`: Stores the content itself. Can be challenging to implement the filtering.
* `Squidex.Domain.Apps.Read.History.Repositories.IHistoryEventRepository`: Stores basic history events to show them in the UI.
* `Squidex.Domain.Apps.Read.Schemas.Repositories.ISchemaRepository`: Stores the schema, including fields and all settings.
* `Squidex.Domain.Apps.Read.Schemas.Repositories.ISchemaWebhookRepository`: Stores the webhook configurations.
* `Squidex.Domain.Apps.Read.Schemas.Repositories.IWebhookEventRepository`: Stores the webhook events, like an internal job queue.

## Command Handlers

Command handlers are used to handle commands. They can be compared with ASP.NET Core Middlewares and run in a pipeline.

    namespace Squidex.Infrastructure.CQRS.Commands
    {
        public interface ICommandHandler
        {
            Task HandleAsync(CommandContext context, Func<Task> next);
        }
    }

They accept two parameters. The first is the command context, that also includes a reference to the command. The next is a function to call the next command handler. Typical use cases are changes to one domain object, for example default fields for new schemas.

### Example 1: Handle command

If you can accept the command, handle it and call `Complete()`.

    class MyHandler : ICommandHandler
    {
        public async Task HandleAsync(CommandContext context, Func<Task> next) 
        {
            if (context.Command is MyCommand myCommand)
            {
                await Handle(myCommand);
                context.Complete();
            }
            else
            {
                await next();
            }
        }
    }

### Example 2: Measure Performance

    class MyHandler : ICommandHandler
    {
        public async Task HandleAsync(CommandContext context, Func<Task> next) 
        {
            var watch = Stopwatch.StartNew();

            try
            {
                await next();
            }
            finally
            {
                watch.Stop();

                Log(watch);
            }
        }
    }

### Example 3: Enrich Command

    class MyHandler : ICommandHandler
    {
        public async Task HandleAsync(CommandContext context, Func<Task> next) 
        {
            if (context.Command is UserCommand userCommand)
            {
                userCommand.UserId = await GetUserIdAsync();
            }

            await next();
        }
    }

## Event Consumers

Event consumers are invoked when new events are created or when an event consumer is restarted and old events are replayed. You should not raise new events, but of course you can create a new events.

    namespace Squidex.Infrastructure.CQRS.Events
    {
        public interface IEventConsumer
        {
            // The name of the event consumer to display in the UI.
            string Name { get; }

            // Filter events by the stream name. Use regular expressions.
            string EventsFilter { get; }

            // Will be invoked when the event consumer is restarted.
            Task ClearAsync();

            // Will be called for each new or replayed event.
            Task On(Envelope<IEvent> @event);
        }
    }