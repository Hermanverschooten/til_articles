visible
-- TITLE --
Let's talk
-- TAGS --
elixir
pub_sub
clustering
-- TLDR --
When we have multiple apps in the same project space, we sometimes want them to talk to each other, Elixir makes this easy.
-- CONTENT --
# Lets' Talk

As is common in many environments, a lot of my projects consist of more than 1 application, usually multiple Phoenix applications.
Sometimes it is necessary for one application to notify the other(s) that something has happened, eg a record changed.
To facilitate this, Elixir offers us a few nice libraries built upon the Erlang networking core.

The most basic example, as seen in the Elixir [guides](https://elixir-lang.org/getting-started/mix-otp/distributed-tasks.html), is connecting 2 nodes and running some code on the other node.
But that is not what we are talking about here. What we want is a simple mechanism to have the nodes subscribe to a topic and listen for notifications.
Sounds familiar? Yes I am talking about PubSub or Publish and Subscribe, offered to us by `Phoenix.PubSub`

A small example, open iex:
```elixir
Mix.install([{:phoenix_pubsub, "~> 2.1"}])
```
This will install, `Phoenix.PubSub` in our current iex session.

```elixir
Phoenix.PubSub.Supervisor.start_link(name: :my_pubsub)
Phoenix.PubSub.subscribe(:my_pubsub, "test")
```
Start the PubSub supervisor, and give it a name, next subscribe to a topic.

```elixir
Phoenix.PubSub.broadcast(:my_pubsub, "test", %{a: 1})
```
Now broadcast something on this pubsub and topic, after flushing we can see the `message` we received.

```elixir
flush
%{a: 1}
:ok
```

So we can now publish and subscribe within this session, not too useful... yet.

## Make it a project

```elixir
mix new talkies --sup
cd talkies
```

Edit `mix.exs` and add `Phoenix.PubSub` to it.

```elixir
   ...
  defp deps do
    [
        {:phoenix_pubsub, "~> 2.1"}
    ]
  end  
  ...
```

And start it with the app in `application.ex`

```elixir
  ...
  def start(_type, _args) do
    children = [
      {Phoenix.PubSub, name: :my_pubsub}
    ]
  ...
```

Get the dependencies and start the app.
```elixir
mix deps.get
iex -S mix
```
If you repeat the `subscribe` and `broadcast`, you will see this already works.

## Make 2 nodes talk

So now that we have this little `app`, let's use it to make 2 versions of our app talk to each other.

In 2 seperate terminals start the app with a short name, `iex --sname app1 -S mix` and `iex --sname app2 -S mix`.
You will notice the `iex` prompt now includes the node name, `app1@computername`.
We will do our subscribe in `app1`, do a publish in `app2`, `flush` in `app1`, and... nothing happened ???
We have started the apps with networking enabled, but we haven't yet connect them.
In either app, do `Node.connect(:"app2@computername")`, where you use the name of the other node as seen in the iex prompt, this name is given as an `atom`.
Now do the broadcast again and flush, yes, we have received our message. Great!


So what have we learned so far?
* If you start your app with the `--sname`, you can connect to other nodes on the same computer.
* If you start a `Phoenix.PubSub` in multiple nodes, with the **same name**, they automically are connected.
* We have not seen this in our example, but all nodes need to use the same erlang cookie! See the above mentioned guide for more info.

## Cluster the apps

While using `Node.connect/1` is fun in an example, in real life it is not that useful.
There is however an excellent library that easily allows us to connect our apps, `libcluster`.
Add `{:libcluster, "~> 3.3}` to your dependencies and `mix deps.get`.

Open `application.ex` and add the following child, `{Cluster.Supervisor, [topologies, [name: Talkies.ClusterSupervisor]]}`.
`libcluster` uses what they call a `topology` to specify how you want your nodes to connect, this can be with a fixed host list and `epmd`, or something like kubernetes, rancher, etc.
We will use the `Cluster.Strategy.Gossip` strategy, this requires almost no configuration on our part and works nicely on the same host, or even within the same network.
Add the topology to the `start` method before the `children`;
```elixir
    topologies = [
      talkies: [
        strategy: Cluster.Strategy.Gossip,
        secret: "my_secret_conversations"
      ]
    ]
```
Only nodes knowing the `secret` can talk to each other. Start both apps again.
After starting the second app, you should notice a message in your console saying `10:19:18.074 [info] [libcluster:talkies] connected to :app1@computer`.
Our nodes have found each other and connected, you can see this with `Node.list/0`.
Se if we subscribe and broadcast, this will already work, nice.
In a `Phoenix` app you would do the same, add the dependency, your topology and add the `Cluster.Supervisor` to the list of children.
Once you do this and you start 2 instances of your app (using `--sname`) channels, ... will work across the instances.
You can subscribe to broadcasts in your live view pages, or use a `GenServer`, ...

```elixir
defmodule Talkies.SubscriberServer do
    use GenServer

    def start_link(opts \\ []) do
        GenServer.start_link(__MODULE__, opts, name: __MODULE_)
    end

    def init(_) do
        Phoenix.PubSub.subscribe(Talkies.PubSub, "notices")
        {:ok, %{}}
    end

    def handle_info(message, state) do
        # Do something with the message
        {:noreply, state}
    end
end
```

## Going into production

We had some fun playing with our nodes in development, but at a certain moment of time we need to deploy to production.
I deploy all my apps on my own servers using LXC containers, so I am going to show you how I do this here.

First we are going to move our topology to our config files and get it in `start` with `topologies = Application.get_env(:talkies, :topology)`.

```elixir

config/dev.exs

  config :talkies,
    topology: [
      talkies: [
        strategy: Cluster.Strategy.Gossip,
        secret: "my_secret_conversations"
      ]
    ]

config/test.exs

  config :talkies, topology: []

config/runtime.exs

  topology_hosts =
    System.get_env("PARTNERS") ||
      raise """
      environment variable PARTNERS is missing, no cluster will be active
      """

  config :talkies,
    topology: [
      talkies: [
        strategy: Cluster.Strategy.Epmd,
        config: [
          hosts: topology_hosts |> String.split(",") |> Enum.map(&String.to_atom/1)
        ]
      ]
    ]
```
To make it easy to add aditional nodes I opted to put them in an environment variable, seperated by commas.
In my systemd unit file, I already had the `ERLANG_COOKIE` set, but I ensure it is the same for all apps that want to talk to each other.
I then add the `PARTNERS`:

```elixir
  Environment=ERLANG_COOKIE=A_very_super_secret_cookie_that_noone_can_guess
  Environment=PARTNERS=app2@example.org,app3@example.org
```

Note that we use long names in production and that `epmd` should be reachable on all hosts.
On the topic of long names, something to consider, I was deploying 2 new apps a couple of days back in the same container and ran into a snag.
I first set `RELEASE_DISTRIBUTION` to `sname`, to use short names, same host, why need long names, `api@mdx-staging`, but that didn't work. 
The erlang networking refused to find the others. So I set it back to `name`, and tried `api@localhost`, this failed too as `localhost` is not a valid long name, duh!
Setting it to the full resolvable hostname would not work because this resolved to an IPv6 address and epmd while listening on `::4369` refused to connect.
So I resolved to point `localhost.example.org` to `127.0.0.1` in my `/etc/hosts` file and then set `PARTNERS=api@localhost.example.org`, that did work!

So don't forget to set the `PARTNERS` environment variable to include all other nodes in each app, do not inlude itself.

When we start the apps now, you should see nodes connect in the logs, if you see messages saying it was not able to connect, check your `PARTNERS` and `epmd`.
This is the most challening part.

## Conclusion

Aside from the `epmd` troubles, it is quite easy to add inter-app communication in Elixir.
But why should you, would you?
I mostly use it to inform one of the other apps something has changed, say you have a customer dashboard and a back-office dashboard in 2 seperate apps.
When a customer places an order, you can broadcast it and do something useful in the back-office app, process the order, send mail, sms, ...
Or in one of my projects, I have one app that talks to a number of `Nerves` devices over `Phoenix` channels, the other app has a dashboard that displays realtime information from the devices, and can send back commands.

Don't forget if you are using multiple different `Phoenix` applications to have them all use the same name for the `PubSub`, by default it will be `YourApp.PubSub`!


**Tip!** Although `epmd` will not allow communication without the correct `ERLANG_COOKIE` it's better to restrict access to all but the `PARTNERS`.

