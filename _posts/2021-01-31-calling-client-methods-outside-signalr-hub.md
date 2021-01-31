---
layout: post
title: Calling client methods outside SignalR Hub class
date: '2021-01-31T12:10:00-07:00'
author: Usein Mambediiev
tags:
- notifications
- rpc
- signalr
---

***Code for this article could be found [here](https://github.com/usemam/signalr-experiments/tree/main/NotificationsExample).***

### Agenda
I'd like to note that the goal of this article is to show one of the possible ways of calling ASP.NET Core SignalR clients outside of Hub class. The main lesson here is that you don't want to reference Hub instance from your app's code - [Hubs are transient](https://docs.microsoft.com/en-us/aspnet/core/signalr/hubs?view=aspnetcore-5.0#create-and-use-hubs). The same restriction also applies to old SignalR version - [see here](https://docs.microsoft.com/en-us/aspnet/signalr/overview/guide-to-the-api/hubs-api-guide-server#transience).

### Domain
The code below implements a use-case of sending arbitrary notifications to all clients connected to a SignalR backend.

Two of our basic interfaces will be
```cs
public interface INotificationSource
{
    IEnumerable<NotificationMessage> GetNotifications();
}
```
and 
```cs
public interface INotificationTransport
{
    Task Notify(IEnumerable<NotificationMessage> messages);
}
```
where *INotificationTransport* is an abstraction for a layer that delivers messages to clients - SignalR in our case.

To connect these two together, we need a third part that will periodically fetch new notifications and send them down the pipe. Something like this:
```cs
public class NotificationConnector
{
    private readonly INotificationSource source;
    private readonly INotificationTransport transport;

    public NotificationConnector(
        INotificationSource source, INotificationTransport transport)
    {
        this.source = source;
        this.transport = transport;
    }

    public Task Start(CancellationToken cancellationToken)
    {
        return Task.Run(() => RunLoop(cancellationToken));
    }

    private async Task RunLoop(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            IEnumerable<NotificationMessage> messages = source.GetNotifications();
            if (messages.Any())
            {
                await transport.Notify(messages);
            }

            await Task.Delay(TimeSpan.FromSeconds(5), cancellationToken);
        }
    }
}
```

### Enter SignalR
Hub
```cs
public class NotificationHub : Hub
{
}
```
*INotificationTransport* implementation
```cs
public class SignalrNotificationTransport : INotificationTransport
{
    private readonly IHubContext<NotificationHub> hubContext;

    public SignalrNotificationTransport(IHubContext<NotificationHub> hubContext)
    {
        this.hubContext = hubContext;
    }

    public Task Notify(IEnumerable<NotificationMessage> messages)
    {
        string payload = JsonConvert.SerializeObject(messages);
        return hubContext.Clients.All.SendAsync("Notify", payload);
    }
}
```
Note that *IHubContext* instance is injected into a constructor. Again - you don't want to keep a reference to *NotificationHub* in your code, because it has transient lifetime and you won't be able to change it. Instead, you should work with *IHubContext* whenever you need to interact with SignalR outside of hubs.

### Wiring things up
To connect things together
- register your dependencies in DI
```cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<INotificationSource, RandomNotificationSource>();
    services.AddSingleton<INotificationTransport, SignalrNotificationTransport>();
    services.AddSingleton<NotificationConnector>();

    services.AddSignalR();
}
```
- call *Start()* on NotificationConnector instance
  
  There are different options here - you can use background service for example, but I chose easier and more dirty way to do this.
```cs
public void Configure(
    IApplicationBuilder app, IWebHostEnvironment env, NotificationConnector connector)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseRouting();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapHub<NotificationHub>("/notificationHub");
    });

    connector.Start(default(CancellationToken));
}
```