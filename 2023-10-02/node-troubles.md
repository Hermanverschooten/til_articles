visible
-- TITLE --
Node troubles
-- TAGS --
elixir
clustering
-- TLDR --
Using a cluster with Elixir is so easy, but sometimes it is not... a tale of a couple of lost hours.
-- CONTENT --
# Node troubles

As most of my readers know I tend to use clustering quite often in my setups.
I run a hotspot business called [GratWiFi](https://www.gratwifi.eu) where the backend exists of several Elixir applications.
Up until a few weeks ago these were 3 Phoenix apps that talk to each other over `Phoenix.PubSub` using `libcluster` to easily interconnect.
Round about the time I upgraded these apps to the latest versions of OTP/Elixir I added a 4th app called `wg_man`.
This new app manages VPN connections for my hotspots using `WireGuard`, replacing `OpenVPN`, it allows setting up the tunnels and managing the IP addresses on both ends.
No trouble there, everything working as it should.

Until some time later I did a deploy of my API server, the dashboards for our customers and partners were no longer updating!
Why? What is happening?  I ssh-ed into the API server, used the `app remote` script, only to notice that `Node.list()` returned an empty list.
Checking on the other servers, they had the other apps except that api in theirs, huh?
I restarted the API server, checked the `Node.list()`, yes we are back, or not... a couple of seconds later, no more connections.
I restarted again, yes, yes, yes, the nodes stay up.

Time to check the logs, my logs in production are JSON structured. This is the log for when it happened again after deploying the dashboard app yesterday.

```elixir
Oct  1 10:20:06 api gratwifi[246545]: {"message":"'global' at node :\"gratwifi@api.xxx\" disconnected node :\"dashboard@dashboard.xxx\" in order to prevent overlapping partitions","time":"2023-10-01T10:20:06.101Z","severity":"WARNING"}
Oct  1 10:20:06 api gratwifi[246545]: {"message":"'global' at node :\"gratwifi@api.xxx\" disconnected node :\"pashboard@partner.xxx\" in order to prevent overlapping partitions","time":"2023-10-01T10:20:06.101Z","severity":"WARNING"}
Oct  1 10:20:13 api gratwifi[246545]: {"message":"'global' at :\"gratwifi@api.xxx\" failed to connect to :\"wg_man@xxx\"\n","time":"2023-10-01T10:20:13.090Z","severity":"WARNING"}
```

What the? overlapping partitions? What is going on? Google to the rescue, a [Stack overlow](https://stackoverflow.com/questions/73567169/distributed-erlang-how-does-the-prevention-of-overlapping-partititions-algor) article explains a lot, but not what is happening and why.
At first I thought it might be happening because of the way I start my apps, I first run migrations then start the app for real. Could it be that the cluster was starting during the migration and immediately stopping again, causing the up/down messages to overlap and cause this?
Ok, remove the `ExecStartPre` from my systemd unit and add `{Task, &migrate/0}` to the list of children in `Application.start/2` and run the migrations.

```elixir
defp migrate do
  if Application.get_env(:api, :run_migrations, false) do
    Api.Release.migrate()
  end
end
```

This combined with setting the flag to `true` in `config/runtime.exs` allows me to run my migrations just in production.

I deployed, restarted and yes all nodes were available. Solved!

That was until yesterday when I deployed the dashboard and it happened again. - Please insert appropriate swear words -

Can I avoid this partitioning thing? Google, ladidadida, yes, you can, add a line to the systemd unit

```elixir
Environment=ELIXIR_ERL_OPTIONS='-kernel prevent_overlapping_partitions false'
```

To be sure, I asked on the Elixir General Slack, if this is a good idea. One of the seemingly always present Elixir masters (@di4na) immediately responded.

> @di4na: yes but beware this can leave your cluster in unstable states where the data reported is not true

So maybe not such a good idea, we talked back and forth, got a lot of suggestions, and I tried to forcefully stop the node's connections on shutdown by adding to `application.ex`:

```elixir
defmodule Api.Application do
  ...

  def stop(_state) do
    :global.disconnect()
    :ok
  end
```

According to the Erlang documentation, this disconnects this node from all others. At first I used `Node.disconnect/1`, but the docs state that to the other nodes it appears as if this node had crashed.

Deploy, test, hmm still happening. Even stranger, starting and stopping different apps yields different results, until I noticed that I could reproduce this issue with 1 app! the `wg_man` app I added last.
Going into the remote console of this app I tried to manually connect to the API server, `false`. This is strange, can I resolve the name from the command line, yes, no probs, I can ping the host.
Hmm, and some more hmm, and a lot of time later, I found the reason why. 

**Oh my God, how stupid can I be?**

Somewhere last year, I got a number of error messages from `EPMD` on the API server, someone was trying to access it from the outside.
So I added some firewall rules to my `ProxMox` instance to restrict access only to the subnet in which my servers reside.
These rules are not visible within the instance, that's why even though I had checked with `iptables` I did not remember.

So what was happening?  `wg_man` restarts and tries to connect to it's list of partners, but it can connect to both dashboard servers, but not to the API server.
The API server and the dashboard server can connect to each other, that's why it sometimes seemed to work. Now I still don't know what triggers the partition checking mechanism in OTP25+, but after adding the `wg_man` server to the rules the problem has gone.

## Protecting other servers

Yesterday afternoon, after I had finally found the problem, I was reminded by @cdegroot on Slack, that you should always protect `EPMD`.

> @cdegroot: I'm not sure what you're trying to do but you really should run all clustered nodes in a single security/networking domain.

The fact that both dashboards could talk to `wg_man` was because they had no firewall rules in place, this I resolved this morning.

And it made me think of other apps I have running in production, all of which are setup as I explained in one of my [previous posts](https://til.verschooten.name/til/2022-09-21/how-i-deploy-my-phoenix-apps).
Checking I see that `EPMD` binds to all addresses on the host, both IPv4 and IPv6 on port 4369.
The Erlang documentation has an environment variable to specify on which addresses `EPMD` should bind.
```elixir
Environment=ERL_EPMD_ADDRESS=127.0.0.1
```
After `systemctl daemon-reload` and restarting the app, `netstat -anp | grep 4369` shows me that it only binds to `127.0.0.1:4369`, success.
But now my script no longer works, it can no longer connect to the running application.
After some trial and error, I needed to make these changes.
```elixir
Environment=RELEASE_NODE=api@127.0.0.1
```
And change the script to:

```elixir
#!/usr/bin/env sh
export APP=$(basename $0)
export RELEASE_NODE=$APP@127.0.0.1
export RELEASE_COOKIE=$(grep -oe "RELEASE_COOKIE=.*" /lib/systemd/system/$APP.service | cut -d'=' -f2)
export RELEASE_DISTRIBUTION=name
export ERL_EPMD_ADDRESS=127.0.0.1
/opt/$APP/bin/$APP $@
```
Be sure to use the IP `127.0.0.1`, I first tried with `localhost` but that's rejected because it is not a _FQDN_.

&#x26A0; Only bind to the local machine if you are not running a cluster on multiple machines.


A Sunday well spent! And thanks @linus to prompt me to write this.
