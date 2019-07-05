---
title: "Writing Slackbots With Goroutines"
date: 2018-07-12T13:20:19-04:00
draft: False
type: "post"
---

It all started with a meme.

This one involved created an insult from everybody’s favorite abusive television chef, Gordon Ramsay, by picking a value from three columns.

- First, you take your month of birth and match it up with an aggressive statement. So let’s say you were born in November, your statement would be “Get a Grip!”.
- Then take the first letter of your first name and do the same. If your name is Sam, then the next part would be “You Proud”.
- And finally, take the first letter of your last name and find the associated value in the third column. If your last name is Quixote, the final part would be “Gremlin!”

Put together, our insult for Sam Quixote, born in November, would be “Get a Grip! You Proud Gremlin!”.

Immediately, I thought this might make for a bit of good fun as a slack bot. I’ve played around with the options here to make our friend Gordon dramatically more supportive instead of the unfortunate verbal tirades. Our friend Sam’s message from Gordon is now this:

{{< figure src="/posts/slackbots/regalYoda.png" width="50%" >}}

This article outlines that whole process and highlights some of the concurrency features of golang. If you’re interested in a compelling alternative to the callbacks and promises of JS, read on!

Let’s first setup a slack bot for your team:

- Go to https://your-slack-team.slack.com/apps/new
- Name your bot and add it to slack. I’ve called ours Gordon.
- In “Integration Settings”, record the API Token.
- Make a little profile image for adorable robotic Gordon:

Now we can get to the actual code!

We’re going to use a great go client library for slack called… slack, so we’ll install that:

{{< highlight bash "" >}}
go get github.com/nlopes/slack
{{< / highlight >}}

This client library interacts with Slack’s Real Time Message API (RTM) (https://api.slack.com/rtm). It’s a very neat API that’s WebSocket-based, so we’ll receive events from slack’s backend as soon as they occur. Let’s start the library and make sure we can see events flowing.

{{< highlight golang "" >}}
main

import (
    "fmt"
    "log"
    "os"
    "strings"

    "github.com/nlopes/slack"
)
func main() {
  api := slack.New("<API-TOKEN-SECRET-SAUCE")
  logger := log.New(os.Stdout, "slack-bot: ", log.Lshortfile|log.LstdFlags)
  slack.SetLogger(logger)
  api.SetDebug(true)

  rtm := api.NewRTM()
  go rtm.ManageConnection()

  for msg := range rtm.IncomingEvents {
        fmt.Println("Event Received: %s\n", msg.Data)
  }
}
{{< / highlight >}}

The guts of this are what’s called a goroutine in golang, which establishes and manages our WebSocket connection to Slack:

{{< highlight golang "" >}}
rtm.ManageConnection()
{{< / highlight >}}

This little gem will also handle reconnecting if your connection fails for some reason.
We can then receive events from the `rtm` object's `IncomingEvents` channel. For now, we're just going to print out the `Type` field of any events we're receiving.

{{< highlight golang "" >}}
for msg := range rtm.IncomingEvents {
  fmt.Println("Event Received: ", msg.Type)
}
{{< / highlight >}}

This is a great example of the simplicity of golang’s concurrency primitives. Instead of something like a javascript callback, we set up a loop which will await all events on the channel. It’s a very synchronous-looking piece of code, but it’s actually describing two sequences executing separately. One is the Slack client library’s internal WebSocket processing, which is placing events on the IncomingEvents channel. The other is our implementation's loop, which is awaiting those events and performing some action when they're received.

Channels, goroutines, and go’s other concurrency primitives are based on ideas from CSP research (Communicating Sequential Processes). A core tenet of go is this:

> Do not communicate by sharing memory; instead, share memory by communicating.

So go attempts to hide more complex ideas like semaphores and memory fences, leaving you free to focus on events flowing through channels to loops receiving messages.

In contrast, Javascript doesn’t execute across multiple threads, so it’s not even possible to share memory. There are no semaphores in happy go lucky JS-land because everything executes in an essentially atomic manner. You can run multiple node processes in a cluster sort of system, but getting them to interact and properly handle IPC can be challenging. Go, on the other hand, is built for this. It provides users with a rich set of tools to manage applications that maximally use available resources.

Back to our favorite chef-bot!

Running this should now briefly spam your terminal. Note that I’ve removed sensitive information and replaced it with REDACTED.

{{< highlight bash "" >}}
run main.go
{{< / highlight >}}

What’s this doing? We check all incoming message events for when our bot is tagged with “@”, and then we send a message to the same channel where that message came from. The empty default case there will allow our channel to receive the message and ignore all other event types sent on the channel without blocking.

Running this results in something like this:

{{< highlight bash "" >}}
Received:  connecting
slack-bot: 2018/02/12 16:53:45 slack.go:123: Starting RTM
slack-bot: 2018/02/12 16:53:45 <autogenerated>:1: parseResponseBody: {REDACTED}
slack-bot: 2018/02/12 16:53:45 slack.go:130: Using URL: wss://REDACTED
slack-bot: 2018/02/12 16:53:45 slack.go:123: Dialing to websocket on url wss://REDACTED
slack-bot: 2018/02/12 16:53:46 slack.go:123: RTM connection succeeded on try 1
Event Received:  connected
slack-bot: 2018/02/12 16:53:46 slack.go:130: Incoming Event: {"type": "hello"}
Event Received:  hello
{{< / highlight >}}

Alrighty! So we've got a realtime connection to Slack's bot API and we can start doing interesting things.

We'll now check for the type of message, and post a very ragey message from our Gordon Ramsay bot if somebody is tagging the bot in a message. Inside our loop, we'll add the following snippet:

{{< highlight golang "" >}}
switch ev := msg.Data.(type) {
case *slack.MessageEvent:
  botTagString := fmt.Sprintf("<@%s>", rtm.GetInfo().User.ID)
  if !strings.Contains(ev.Msg.Text, botTagString) {
    continue
  }
  rtm.SendMessage(rtm.NewOutgoingMessage("IT'S BURNTTTT!", ev.Channel))

default:

}
{{< / highlight >}}

What’s this doing? We check all incoming message events for when our bot is tagged with “@”, and then we send a message to the same channel where that message came from. The empty default case there will allow our channel to receive the message and ignore all other event types sent on the channel without blocking.

Running this results in something like this:

{{< figure src="/posts/slackbots/believeInYou.png" width="50%" >}}

Now, let’s do something a bit fancier and match what’s in our meme. We need to watch for messages of the form:

“@gordon “

So we’ll take our message and split it into its component parts, sending a message if the user doesn’t send a message with enough parts to it.

{{< highlight golang "" >}}
messageParts := strings.Split(message, " ")
if len(messageParts) != 3 {
    rtm.SendMessage(rtm.NewOutgoingMessage("I Believe in You", ev.Channel))
}
{{< / highlight >}}

Then we have a few very simple lookup tables (simplified for brevity) that we can search through to create a personalized insult, courtesy of our abusive friend Gordon Ramsay. https://gist.github.com/466c45978fe01d9525529dd39278268a

And finally we can send our message:

{{< highlight golang "" >}}
rtm.SendMessage(rtm.NewOutgoingMessage(message, ev.Channel))
{{< / highlight >}}

Running this:

{{< figure src="/posts/slackbots/fullBot.png" width="50%" >}}

Well done Gordon!

You can find a cleaned up version of this code at https://github.com/zachgoldstein/gordonBot. There’s a whole slew of things to do before this would be ready for production, but a few of them are in place in this code. I’ve added a few tests for the guts of this, and instead of hard-coding your API-key you can pass it in via a command line flag. If you’d like to see how what that looks like, definitely check it out.

---

Originally published at x-team.com on February 22, 2018