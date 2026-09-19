visible
-- TITLE --
Blue/Green deployments with Caddy
-- TAGS --
high-availability
caddy
deploy
deployment
phoenix
-- TLDR --
Reducing the downtime of a Phoenix app with a blue/green deployment.
-- CONTENT --
As readers of this site know I spend a reasonable time on improving my deployments of my Phoenix-based apps.
A couple of weeks ago I switched to my first blue/green deployment and have been improving it steadily.

## What is a blue/green deployment?

In a normal deployment we copy our new code to the server, stop the running instance, unpack the new code and restart the server.
This has the negative effect of interrupting our app for a couple of seconds. Phoenix does not just stop, it drains the connections and when you have active liveview (or channel) sessions it can take up to 30 seconds for it to actually stop.
The idea behind blue/green deployments is that you start the new code next to the old one and once it is healthy you switch all traffic over.

## The initial attempt

My initial attempt did just that. I added a `/health` endpoint and updated my `deploy` script.
We unpack our code in a `staging` folder, we keep track of our current version in a file.
Let's say our current active instance is `blue`, so the new one needs to become `green`.
* We remove the current `green` folder, rename our staging to it.
* We start the new `green` instance using `systemd`.
* We wait a set amount of time for it to become healthy.
* We update the active file to point to `green`.
* We update Caddy to use the port of our `green` version, say `4001`, and reload Caddy.
* We stop the `blue` version.
* Success

## The problems arise

I ran this kind of deployment for a number of servers and was quite happy with it. The liveview users would have a small blip during the cut-over but most never noticed.
And life was good... as long as I had a single app on a single Caddy/container with no clustering.

When you cluster your Elixir/Erlang apps a small utility server called `epmd` (Erlang Port Mapper Daemon) is started in which your app registers itself with it's cluster port.
This allows the clustering to let the apps talk to each other.  When you start your app from `systemd` `epmd` is started with your first app deployment and everything is nice.
But as soon as you deploy a new version of the app that started `epmd` trouble is on the horizon.
`systemd` starts your app with a process group and `epmd` belongs to that process group, so what happens when you stop that app instance? Correct `epmd` gets shutdown too, breaking your cluster.

Oh and I forget to tell you about needing different names for your instances for `epmd`, so your clustering mechanism needs to know how to handle that, there can be `app_a_blue` and `app_a_green` briefly at the same time.

## Solving the `epmd` problem

I searched with the help of my reluctant assistant Claude for a durable solution to the `epmd` issue, I was still running single app/container but with 4 apps clustered using `libcluster` and it had already bitten me a couple of times that the apps were no longer talking to each other and my `PubSub` no longer worked.
Claude tried to make add pre/post-scripts to `systemd`, `cron` jobs, you name it.
In the end I resolved it by copying the `epmd` executable out of the latest deployment and giving it its own `systemd` service. I know what you are all thinking, what if there is an update of `epmd`, yes I know that is the catch. But `epmd` is such a small C-program that I do not see this happen often, but it is something I need to keep track of.
This also required me to stop the erlang distribution from trying to start `epmd`, see the `systemd` file below.

## Multiple apps on the same machine

I try to keep to single app/container as much as I can but sometimes this is not possible, think shared resources, ...
So adding another app is just a new `deploy` file with new ports, isn't it? Yes, mostly.
My initial `deploy` script did a find/replace on the `Caddyfile` to switch the ports, this ofcourse broke the `Caddyfile` when I added the second app, sigh regexes... but that was easilly solved.
On this server the app that was installed first has a number of channel connections to external devices, not liveview users, when you update the `Caddyfile` and reload `Caddy` what happens?
Correct all connections are severed for both apps, I did not notice this at first, but read this again a couple of times... I'll wait... for BOTH apps.
So app A is quietly chugging along servicing it's users and app B comes along with a new version. App B starts it's second instance, Caddy is reloaded and all of app A's connections are gone and reconnecting.

## Wrestling with Caddy

Obviously this is not what we want?  Claude to the rescue! Instead of updating the `Caddyfile` and reloading `Caddy`, let's use the `Caddy API` to update the ports dynamically. Great find! Let's do it.
A short while later the scripts have been updated, first deploy succeeds and I am happy or not quite...
This is even worse!
`Caddy` does not like you changing the config using the `API`.
Every change you make to the config is seen as atomic and causes `Caddy` to do a reload.
And we know what a reload does... it terminates all connections.
So going from a single reload, we now are calling the `API` several times to update the ports and each time we do, `Caddy` reloads.
**SIGH**
Then Claude offered to use the load balancer functionality of `Caddy` and put both ports in it, this worked but caused such a lot of log messages as `Caddy` is constantly checking for the offline port I soon searched for a cleaner solution.

## The final solution

So after fixing the `epmd` issue and the naming conflicts, have we come to a dead-end?
Not quite, `Bandit` (as does `Cowboy`) allows us to use a `unix socket file` instead of a `TCP Port`.
We update our `Phoenix` runtime config to use `{:local, socket_path}` which we get from the enviroment in `runtime.exs`, we link a  `/var/run/<app>/current.sock` to it and use that in the `Caddyfile`.
This allows us to keep our `Caddyfile` untouched during a new deployment, `Caddy` reopens the socket file on each connection so when we relink it during deploy all new connections go to the new file, the new app.

Were all issues resolved by going to the `unix socket file`? Not quite, not all libraries like the `{:local, path}` tuple when they expect an IP address.
Mostly these can be solved by using `remote_ip` or adding a small `plug` like this one:
```elixir
defmodule App.Plugs.NormalizeSocketRemoteIp do
  @moduledoc "Rewrites a unix-socket peer (`{:local, path}`) to loopback."
  @behaviour Plug

  @impl true
  def init(opts), do: opts

  @impl true
  def call(%Plug.Conn{remote_ip: {:local, _path}} = conn, _opts) do
    %{conn | remote_ip: {127, 0, 0, 1}}
  end

  def call(conn, _opts), do: conn
end
```

## The files

**config/runtime.exs**
```elixir
  http_options =
    case System.get_env("SOCKET") do
      nil ->
        [ip: {0, 0, 0, 0, 0, 0, 0, 0}, port: String.to_integer(System.get_env("PORT") || "4000")]

      socket_path ->
        [ip: {:local, socket_path}, port: 0]
    end
  config :app, AppWeb.Endpoint,
    url: [scheme: "https", host: "app.example.org", port: 443],
    http: http_options,
    secret_key_base: secret_key_base,
    server: true,
    ...
    ]
```

### deploy.sh

```elixir
set -euo pipefail

APP="$(basename "$(dirname "$(readlink -f "$0")")")"
BASE="/opt/$APP"
STATE_FILE="$BASE/active_slot"
STAGING_DIR="$BASE/staging"
CURRENT_SOCKET="/run/$APP/current.sock"
HEALTH_RETRIES=30
HEALTH_INTERVAL=2

ACTIVE="$([[ -f "$STATE_FILE" ]] && cat "$STATE_FILE" || echo none)"
if [[ "$ACTIVE" == "blue" ]]; then TARGET=green; else TARGET=blue; fi
TARGET_SOCKET="$(grep -E '^SOCKET=' "$BASE/$TARGET.env" | cut -d= -f2)"

echo "==> $APP active=$ACTIVE -> $TARGET (socket $TARGET_SOCKET)"
rm -rf "$BASE/$TARGET"
mkdir -p "$BASE/$TARGET"
tar -zxf "$STAGING_DIR/$APP.tar.gz" -C "$BASE/$TARGET"

systemctl start "$APP@$TARGET"

echo "==> waiting for $TARGET /health on $TARGET_SOCKET"
for i in $(seq 1 "$HEALTH_RETRIES"); do
  if curl -sf --unix-socket "$TARGET_SOCKET" "http://localhost/health" >/dev/null 2>&1; then
    echo "==> health OK"
    break
  fi
  if [[ $i -eq $HEALTH_RETRIES ]]; then
    echo "==> health FAILED after $((HEALTH_RETRIES * HEALTH_INTERVAL))s, rolling back"
    systemctl stop "$APP@$TARGET"
    exit 1
  fi
  sleep "$HEALTH_INTERVAL"
done

echo "==> switching Caddy to $TARGET_SOCKET"
ln -sfn "$TARGET_SOCKET" "$CURRENT_SOCKET"

systemctl enable "$APP@$TARGET" >/dev/null 2>&1 || true
echo "$TARGET" > "$STATE_FILE"

if [[ "$ACTIVE" != "none" ]]; then
  echo "==> stopping old slot $ACTIVE (only $APP's own live connections drop)"
  systemctl disable "$APP@$ACTIVE" >/dev/null 2>&1 || true
  systemctl --no-block stop "$APP@$ACTIVE" || true
fi

echo "==> $APP deploy complete. active=$TARGET"
```

### systemd file
```elixir
# /etc/systemd/system/app@.service
[Unit]
Description=app (%i)
After=network.target
After=epmd.service
Requires=epmd.service
[Service]
TimeoutSec=120
User=root
# Group=caddy (not root) so any socket file this instance binds (see SOCKET
# in %i.env) is group-owned by caddy from the moment it's created
Group=caddy
UMask=0007
# Shared by both slots so the persistent current.sock symlink and whichever
# slot's not-yet-active socket both survive the other slot's restarts.
# RuntimeDirectoryPreserve=yes is load-bearing, not cosmetic: without it,
# systemd removes /run/app the moment ANY unit instance that declares
# it stops -- including the slot being retired at the end of a normal
# deploy -- even while the other slot is still actively using it.
RuntimeDirectory=app
RuntimeDirectoryPreserve=yes
EnvironmentFile=/opt/app/env
EnvironmentFile=/opt/app/%i.env
# Bandit/Thousand Island does not unlink a pre-existing path before bind()ing
# a unix socket -- it crashes with :eaddrinuse. Every slot leaves its own
# socket file behind when it stops (nothing removes it), so without this the
# *next* time this same slot starts, every deploy after that would fail
# deterministically. Also confirmed live during dashboard's migration.
ExecStartPre=/bin/rm -f /run/app/%i.sock
ExecStart=/opt/app/%i/bin/app start
ExecStop=/opt/app/%i/bin/app stop
# Reboot safety net: deploy's own `ln -sfn` is what normally repoints
# current.sock, but a reboot restarts whichever slot is `enabled` without
# going through deploy. If that slot is the one active_slot names, relink
# it so Caddy has something to dial even without a deploy having run.
ExecStartPost=/bin/sh -c '[ "$(cat /opt/app/active_slot 2>/dev/null)" != "%i" ] || ln -sfn /run/app/%i.sock /run/app/current.sock'
Restart=on-failure
[Install]
WantedBy=multi-user.target
```

### env file
```elixir
LANG=C.UTF-8
LC_CTYPE=C.UTF-8
ELIXIR_ERL_OPTIONS="+fnu -start_epmd false"
HOME=/root
SECRET_KEY_BASE=...
DATABASE_URL=ecto://...
ECTO_IPV6=false
RELEASE_COOKIE=...
PHX_SERVER=true
```

### blue/green env files
```elixir
RELEASE_DISTRIBUTION=name
RELEASE_NODE=app_blue@127.0.0.1
SOCKET=/run/app/blue.sock
```

### Caddyfile
```elixir
app.example.org {
        reverse_proxy unix//run/app/current.sock {
                header_up X-Real-IP {remote_host}
        }
        log {
                output file /var/log/caddy/app.access.log {
                        roll_size 100MiB
                        roll_keep 10
                }
                format json
        }
}
```
### epmd service
```elixir
# /etc/systemd/system/epmd.service
# Erlang Port Mapper Daemon (Independent)
[Unit]
Description=Erlang Port Mapper Daemon (Independent)
After=network.target
Documentation=man:epmd(1)

[Install]
WantedBy=multi-user.target

[Service]
Type=forking
ExecStart=/usr/local/bin/epmd -daemon
Restart=always
# Do not wait before restarting if it crashes
RestartSec=0s

# --- SECURITY & ISOLATION ---
DynamicUser=true

# Prevent it from gaining root privileges
NoNewPrivileges=true
# Make the root filesystem read-only to epmd
ProtectSystem=strict
# Give it a private, isolated /tmp space
PrivateTmp=true

# --- LOGGING ---
# Send all stdout/stderr directly to the system journal
StandardOutput=journal
StandardError=journal
SyslogIdentifier=epmd

[Install]
WantedBy=multi-user.target
```
