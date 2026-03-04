visible
-- TITLE --
Eliminating EPMD from my Elixir cluster
-- TAGS --
elixir
erlang
epmd
clustering
deployment
-- TLDR --
How I replaced EPMD with a static module to fix restart crashes in my distributed Erlang cluster.
-- CONTENT --
# Eliminating EPMD from my Elixir cluster

I run three Elixir apps on my mail server — CountryCheck, MailMan and Webmail — clustered together via distributed Erlang so they can share PubSub messages. This has been working fine, except for one annoying problem: restarts.

## The problem

When a service restarts, EPMD (the Erlang Port Mapper Daemon) sometimes still holds the old node name registration. The new instance tries to register the same name, finds it "in use", and crashes:

```
the name country_check seems to be in use by another Erlang node
```

My workaround was a script called `epmd_wait_free` that polls EPMD for up to 10 seconds waiting for the stale name to clear. Each service had a systemd `ExecStartPre` drop-in calling it. It worked, but it felt like duct tape.

## The idea

EPMD's job is simple: it maps node names to ports. When node A wants to connect to node B, it asks EPMD on B's host "what port is node B listening on?". But if you know the port in advance, you don't need to ask.

The BEAM lets you replace EPMD with a custom module via `-epmd_module`. If every node uses a fixed port, the module just returns that port for any query.

The trick is that all three apps run on the same machine, so they can't all bind to the same port. But Linux routes the entire `127.0.0.0/8` range to loopback. Give each app its own loopback IP and they can all use port 6789:

| App | Loopback IP | Node Name |
|-----|-------------|-----------|
| CountryCheck | 127.0.0.1 | `country_check@127.0.0.1` |
| MailMan | 127.0.0.2 | `mail_man@127.0.0.2` |
| Webmail | 127.0.0.3 | `webmail@127.0.0.3` |

## The implementation

The module is tiny. Drop it in `lib/static_epmd.ex` in each project:

```elixir
defmodule StaticEpmd do
  @dist_port 6789

  def start_link, do: :ignore
  def register_node(_name, _port), do: {:ok, :rand.uniform(3)}
  def register_node(_name, _port, _family), do: {:ok, :rand.uniform(3)}
  def port_please(_name, _host), do: {:port, @dist_port, 5}
  def port_please(_name, _host, _timeout), do: {:port, @dist_port, 5}
  def names(_hostname), do: {:ok, []}
end
```

`port_please` always returns 6789. `register_node` returns a dummy creation value. `start_link` returns `:ignore` because there's nothing to start.

Then three release config files per app. For CountryCheck:

`rel/env.sh.eex`:
```elixir
#!/bin/sh

export RELEASE_DISTRIBUTION=name
export RELEASE_NODE=country_check@127.0.0.1
```

`rel/vm.args.eex`:
```elixir
-start_epmd false
-epmd_module Elixir.StaticEpmd
-kernel inet_dist_listen_min 6789 inet_dist_listen_max 6789
-kernel inet_dist_use_interface {127,0,0,1}
```

`rel/remote.vm.args.eex`:
```elixir
-start_epmd false
-epmd_module Elixir.StaticEpmd
```

MailMan and Webmail get the same files, just with their own IP (`.2` and `.3`).

The `remote.vm.args.eex` is important — without it, running `bin/myapp rpc '...'` would try to start EPMD to connect to the running node.

## The cluster_connect script

The script that connects the nodes on startup needed updating to use the new `@127.0.0.X` addresses instead of `@mail.octarion.eu`:

```elixir
#!/bin/sh
sleep 5
APP="$1"
case "$APP" in
  country_check) PEERS="mail_man@127.0.0.2 webmail@127.0.0.3" ;;
  mail_man)      PEERS="country_check@127.0.0.1 webmail@127.0.0.3" ;;
  webmail)       PEERS="country_check@127.0.0.1 mail_man@127.0.0.2" ;;
esac
for peer in $PEERS; do
  /usr/sbin/"$APP" rpc "Node.connect(:\"$peer\")" 2>/dev/null || true
done
```

## Cleanup

With EPMD out of the picture for the clustered apps, I could remove:
- `/usr/sbin/epmd_wait_free`
- The systemd `ExecStartPre` drop-ins for all three services
- The `RELEASE_NODE` and `RELEASE_DISTRIBUTION` exports from the wrapper scripts (now set in `rel/env.sh.eex`)

## What about the other BEAM apps?

I also run Mailfilter and Greylist as separate BEAM VMs on the same server. They're not in the cluster but they do use distribution for `rpc` and `remote` commands. They still use EPMD the traditional way and are completely independent of the clustered apps on port 6789.

## The result

After deploying all three apps:

```elixir
$ ssh mail "ss -tlnp | grep 6789"
LISTEN  127.0.0.1:6789   # country_check
LISTEN  127.0.0.2:6789   # mail_man
LISTEN  127.0.0.3:6789   # webmail

$ ssh mail "country_check rpc 'IO.inspect(Node.list())'"
[:"mail_man@127.0.0.2", :"webmail@127.0.0.3"]
```

No more EPMD races on restart. No more polling scripts. Just three nodes, each on their own loopback, always knowing where to find each other.


Life can be a dream.
