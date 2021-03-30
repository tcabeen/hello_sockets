# Real-Time Phoenix Notes

This document contains my notes, separated by chapters, beginning with Part I. Powering Real-Time APplications with Phoenix. I won't take any notes for the introduction or chapter 1, which is just a bit of hype for getting into the book.

## 2. Connect a Simple WebSocket

WebSockets are a new standard that came with HTML5. Pages still connect to http initially, but ws:// calls are made within the logic of the page. It makes me wonder why I haven't seen these in my scraping before. They are a way to create a stable connection between server and client.

### Set up a project

This chapter has us setting up a new project called hello_sockets with the following command:
`mix phx.new hello_sockets --no-ecto`
I'm pretty sure no-ecto just means we're rolling without a database on this project.

### Edit app.js to enable WebSockets

We begin by editing hello_sockets/assets/js/app.js to enable the line:
`import socket from "./socket"`
I'm not sure I even remembered to save the file before continuing, but the book says it's required. Onward!

Start up the server with `mix phx.server`
This will commandeer your terminal, so open a separate terminal window for it .. or a separate one to continue.
Then visit [`localhost:4000`](http://localhost:4000) to see the site.

### Researching WebSockets in the browser inspector

This was familiar territory, tracing the site through the Network tab. We filtered on WS to just see the WebSocket traffic. The author continued with information about keeping connections alive and some basics on security.

Beyond WebSockets, we also learned about long-polling in HTTP and saw some pros and cons vs WebSockets.

## 3. First Steps with Phoenix Channels

Here we dive into Channels, which seem like a super dope feature. Thousands of channels are possible across a single connection, and high-performing at the same time. They are also transport-agnostic, so when WebSockets are some day replaced, Channels will work through the new infrastructure.

We also get our first introduction to the hierarchical triumverate of:

1. Phoenix.Socket routes client requests
2. Phoenix.Channel starts up a process for each topic a user connects to
3. Phoenix.PubSub routes messages to and from channels on the server side, permitting the use of multiple nodes

### Sockets

Sockets, the backbones of real-time communication, are modules that implement Phoenix.Socket.Transport behaviors. Phoenix.Socket is great, because it handles both WebSockets and long-polling according to the best practices of each. (We could roll our own Socket.Transport as well.)

A basic functional Socket doesn't require much code at all.

Edit hello_sockets/lib/hello_sockets_web/channels/user_socket.ex to use Phoenix.Socket and create the first channel. Then add functions to def connect() and id().

Next edit hello_sockets/lib/hello_sockets_web/channels/ping_channel.ex to def a join() function without any logic. THen we can def a handle_in() function to respond to ping requests.

Test the connection to this channel by installing wscat and using it to send requests to the app.

`npm install -g wscat` gets us set up

Then, with our server running (so we'll need 2 open Terminal windows)
`wscat -c 'ws://localhost:4000/socket/websocket?vsn=2.0.0'` will connect us to the server.
`["1","1","ping","phx_join",{}]` will connect us to the ping server
`["1","2","ping","ping",{}]` will send the ping request that gets us a response of {"ping":"pong"}. So cute.

Here I was making a ton of typos becuase my mechanical bt keyboard doesn't really do that well. Such a shame. I still insisted on using it, like it was just rusty lol. Fixed the typos and reran things until they worked.
It was a while before I switched over to the cheap Logitech.

We did a little reading on channel error handling and then intentionally crashed the server.
`wscat -c 'ws://localhost:4000/socket/websocket?vsn=2.0.0'` to reconnect
`["1","1","ping","phx_join",{}]` first
`["1","2","ping","ping",{}]` then a nice reply
`["1","2","ping","ping2",{}]` for a channel that doesn't exist
`["1","2","ping","ping",{}]` to see our channel is no longer open
then phx_join and ping again to resolve the matter.

All this time, the terminal window running the server is very helpfully showing debug messages.

### Topics

We learned that "ping" above was called a topic, or a string identifier used to connect to a channel. Topics can be any string, and topic:subtopic formatting is best practice. This makes it possible for a socket to accommodate multiple channels.

Wildcards are allowed as subtopics, making it possible to open a ton of channels. Time for another exercise!

Edit user_socket.ex again to add this channel below "ping":
`channel "ping:*:", HelloSocketsWeb.PingChannel`

`wscat -c 'ws://localhost:4000/socket/websocket?vsn=2.0.0'` Back to our test console!
`["1","1","ping:wild","phx_join",{}]` joins the "wild" subtopic that we permitted with a wildcard
`["1","1","ping:wild","ping",{}]` pong!

Wildcards aren't open-ended. Something like "ping:\*s" would fail to compile.

### More complex subtopic logic

Multiple values are possible in a subtopic. Let's get buckwild with an example that receives 2 integers in the subtopic and checks that the latter is double the former.

user_socket.ex gets a new channel:
`channel "wild:\*", HelloSocketsWeb.WildcardChannel

And a whole new script to handle the logic at:
hello_sockets/lib/hello_sockets_web/channels/wildcard_channel.ex
It gets the same join() and handle_in() functions defined.
It also gets a third function, numbers_correct?()
The ? notation is fascinating to me. I wonder if it suggests that the function should always return a boolean value as a test, or if it could reasonably accommodate more complex logic. Anyway, it splits the numbers on the : character and tests the math. The syntax is not pythonic and includes things like |> and -> ... so I'm looking forward to understanding that.

Then we head back into wscat for testing.
Passing "wild:1:2" is wildly successful.
Passing "wild:1:3" results in a tasteful error.
Passing "wild:a:b" is absolutely catastrophic.
Passing "wild:2:4:6" errors gracefully
The chapter doesn't go into it, but a:b could be handled with type checks. No idea why it's so chill about 2:4:6 tho. If it separates into a, b, and c values, then the test should pass, as only a and b are considered. It would make no sense for it to parse 4:6 into b, and if it did, that should be catastrophic, right? Baffled.

### PubSub

PubSub, we finally learn, is short for publisher/subscriber. Learning to configure it for performance and availability is beneficial, but after that, channels interface with it directly, so we won't need to.

PubSub links the local node to all remote nodes and can broadcast messages anywhere. As long as the nodes can intercommunicate, clients and responses can reach one another through PubSub.

It looks like the standard adapter is called pg2, but a Redis PubSub adapter is available and presumably others.

This was a truly brief exercise, but we used `iex` for the first time. I've been reading about it, and it's similar to when we used mix to start the server, but it leaves us an input so that we can push content from the server to clients (among other things, presumably).

In this exercise, we launched the server with
`iex -S mix phx.server`

Then, on the client terminal, we ran the same 3 commands again.
`wscat -c 'ws://localhost:4000/socket/websocket?vsn=2.0.0'`
`["1","1","ping:wild","phx_join",{}]`
`["1","1","ping:wild","ping",{}]`

Back to the terminal to enter more commands, but if you're taking a few seconds to do it, you can hit that ping command every 20 seconds to prevent a timeout.

From the server's iex(n)> prompt:
`HelloSocketsWeb.Endpoint.broadcast("ping", "test", %{data: "test"})` and a message will appear on the client! Because it's connected to the "ping" topic.
`HelloSocketsWeb.Endpoint.broadcast("other", "x", %{})` but nothing happens on the client, because it isn't connected to the "other" topic.

At the end of this section was an aside about reading source code. 🥰

--- holder ---
Oh my days it's after midnight again.

# TODO

1. Figure out why my ex scripts get a cute Elixir icon but no syntax highlighting
