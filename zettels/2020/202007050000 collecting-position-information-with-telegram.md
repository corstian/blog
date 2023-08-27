---
title: "How to collect location data with Telegram and .NET Core?"
slug: "collecting-position-information-with-telegram"
date: "2020-07-05"
summary: ""
references: 
---

#software-development #dotnet

In the name of prototyping and quick idea validation I have turned to Telegram to validate both the technical, social and emotional sides related to spatiotemporal matching. Telegram is an easy and cheap alternative to building a mobile app for user tracking. Although there is less control over how the data is collected, it is a decent way to test several things out before fully committing to such big project.

In this post I am going to cover the technical aspects of implementing the collection of position information using Telegram. Later on I will write a bit more about the social and emotional aspects that come with fully automatic flight logging over at the Skyhop blog.


## Getting started with Telegram

One of the great things about Telegram is how they offer first class developer support through their API's, something I wish WhatsApp would have offered. There is a whole ecosystem built around, and there are many libraries abstracting the communication bits away and give you the ability to pack a lot of functionality in a relatively short piece of code. [A friend of mine even managed to write a Telegram bot using a single line of code!](https://gist.github.com/jwdb/6e5b4daa360ce0c793dfd48ee765789f)

I have chosen to use the [`Telegram.Bot` package](https://github.com/TelegramBots/Telegram.Bot) to abstract some of the internals away, and therefore it is a dependency for the code I will be presenting in this post.

> In order to figure out how to set up a basic bot I'd recommend you to check out [this article](https://telegrambots.github.io/book/1/quickstart.html).

Telegram has two methods of delivering messages sent to your bot. The first one is webhooks, while the second ones depend on your service making an connection to Telegram. I'll be using the last method because of the better developer experience.

```csharp
class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Hello World!");

        var botClient = new TelegramBotClient("your-bot-token");

        botClient.OnMessage += async (s, e) =>
        {
            
        };

        botClient.OnUpdate += async (s, e) =>
        {
            
        };

        botClient.StartReceiving();
        
        Console.ReadKey();
    }
}
```

At this point I usually do have to make some important decisions:

1. Is it required that the bot autonomously sends users information? If so, what is going to be the data source?
2. How is this bot going to communicate? Does it need natural language pattern recognition, or are we going to provide navigation options?
3. How are we going to keep track of bot users?

> Don't be fooled by the complexity that comes with a communication structure! This is something which becomes mind boggling complex too quickly. Maybe I will write a library to help tackle this problem in the future. [Let me know if you're interested!](https://twitter.com/corstianboerman)


## Giving the bot something to do
Currently the bot does nothing. Though to Telegram it may look like the messages are received, nothing is done with it. Given we're trying to receive position information, let's see whether we can receive a single data point. The message's event arguments contain a `MessageType` variable which tells us the type of message we're dealing with. We're primarily focussing on messages with the `MessageType.Location` enum value.

To give life to your bot it needs to do something with the messages it receives. Doing so can be as easy as this:

```csharp
botClient.OnMessage += async (s, e) =>
{
    try
    {
        if (e.Message.Location != null)
        {
            await botClient.SendTextMessageAsync(
                chatId: e.Message.Chat,
                text: $"Lat: {e.Message.Location?.Latitude}, Long: {e.Message.Location?.Longitude}");
        }
        else
        {
            await botClient.SendTextMessageAsync(
                chatId: e.Message.Chat,
                text: e.Message?.Text ?? "Unrecognized content");
        }
    }
    catch { }
};
```

### Reacting to a live location
The `OnMessage` event handler will only get you so far. Though it works quite decent when sending a single location, and it will be notified initially when sharing a live location, it will not be triggered on subsequent location updates. To do so we would need to look at updates to the original message:

```csharp
botClient.OnUpdate += async (s, e) =>
{
    try
    {
        if (e.Update.EditedMessage?.Location != null)
        {
            await botClient.SendTextMessageAsync(
                chatId: e.Update.EditedMessage.Chat,
                text: $"Lat: {e.Update.EditedMessage?.Location?.Latitude}, Long: {e.Update.EditedMessage?.Location?.Longitude}");
        }
    }
    catch { }
};
```

> Interestingly enough the `OnUpdate` handler also gets triggered when a new message is sent. The difference with real updates however is that the `EditedMessage` property will be empty, and you can retrieve the message contents from the `OriginalMessage` property. This might be helpful to prevent code duplication.

## The bot
All the bot does is wait for messages containing a location, and parrot the location back to the user. Latitude and longitude are also the only values you can read from the location. If you wish to retrieve information regarding the heading, speed or other properties like these you should derive them from the coordinates themselves.

![](/uploads/Telegram_bot_chat_8688b34dd8.png)


### Testing the tracking accuracy

In order to test the accuracy of the reported positions I hopped on my bike and drove along the Eastern Scheldt for a bit. Most interestingly the reported positions can be incredibly accurate with an error smaller than a few meters, but also off by a big margin, like three kilometers. There's no way I managed to swim 6 kilometers within 20 minutes.

![](/uploads/Telegram_Tracking_Inconsistencies_70242a5af0.jpg)

The sampling rate however is pretty low, with an position report about every minute.


## Recap

All in all it's pretty simple to start collecting position information from a Telegram bot. What would be more difficult however is to derive added value from these position updates. Given the sample rate is relatively low, and the margin of error relatively high, there are some challenges to overcome.
