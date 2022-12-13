visible
-- TITLE --
How I deploy my Phoenix apps
-- TAGS --
elixir
deploy
phoenix
-- TLDR --
Most of my Phoenix deplyments follow a similar deployment strategy.
I run them on a LXC Container on ProxMox, using an Ubuntu 20/22 image.
-- CONTENT --
# How I deploy my Phoenix apps

## Prerequisites
What do I need to do when I setup a new deployment?

  - Setup a new Ubuntu container on one of my ProxMox servers.
  - Install Postgres
  - Copy scripts/systemd unit-files
  - create /opt/app folder
  - Add github workflows to my app.
  - Add the necessary secrets to the Github repo.

I'll go over each in more detail below.

_Please replace all instances of *app* with the real name of the application._

### Setup the container
On the ProxMox server I select to create a new container, from the list of available images, I choose the most recent Ubuntu image, currently 22.04 LTS.  For memory I usually set 1GB and HD space depends on what app this will be.

Once the image has been created and the instance started, I ssh into it and do the necessary `apt-get update` and `apt-get dist-upgrade`, before continuing with the installation of Postgresql.

### Install Postgresql
Most of the time the version of Postgresql available on the Ubuntu default apt-repository suffices.  The latest at the moment of writing is 14. 
```elixir
apt-get install postgresql
sudo -u postgres
createuser -P app
createdb -O app app_production
exit
psql -h localhost -U app -W app_production
```
First install Postgresql, then switch to the postgres user and create the role and database. Lastly check that the role really has access.  I have noticed I need to add the `-h localhost` or the connection will fail.

### Copy scripts/systemd unit-files
These are the files I usually copy over.

 - ~/deploy
 ```elixir
#!/usr/bin/env bash
sudo systemctl stop app
cd /opt/app
tar -zxf app.tar.gz
sudo systemctl start app
 ```

 - /usr/sbin/app
 ```elixir
#!/usr/bin/env sh
export APP=$(basename $0)
export RELEASE_NODE=$APP
export RELEASE_COOKIE=$(grep -oe "RELEASE_COOKIE=.*" /lib/systemd/system/$APP.service | cut -d'=' -f2)
export RELEASE_DISTRIBUTION=name
/opt/$APP/bin/$APP $@
 ```

 - /lib/systemd/system/app.service
 ```elixir
[Unit]
Description=app Website
After=syslog.target
After=network.target

[Service]
TimeoutSec=120
User=root
Group=root

Environment=LANG=C.UTF-8
Environment=LC_CTYPE=C.UTF-8
Environment=HOME=/root
Environment=PORT=
Environment=SECRET_KEY_BASE=
Environment=DATABASE_URL=ecto://app:password@localhost/app_production
Environment=RELEASE_DISTRIBUTION=name
Environment=RELEASE_COOKIE=
Environment=PHX_SERVER=true

ExecStartPre= /opt/app/bin/app eval "app.Release.migrate()"
ExecStart= /opt/app/bin/app start
ExecStop= /opt/app/bin/app stop
Restart= on-failure

[Install]
WantedBy=multi-user.target
 ```

The first 2 scripts obviously need to be executable, so `chmod +x` if necessary.
The `deploy`-script is run from the Github action to copy the new release in place and restart the app.
The `app`-script allows me to easily use the _remote iex-console_ of the running app, be ssure to rename it the name of your app, or it will not find the service-file.

The systemd unit-file needs some more adjustments, depending on if it will be running behind an Nginx reverse-proxy or if I will be using [SiteEncrypt](https://hexdocs.pm/site_encrypt/readme.html). Certainly we need to fill in the missing entries, and adjust the database connection info.
Once those are set, we instruct systemd of our changes, and enable the app.
```elixir
systemctl daemon-reload
systemctl enable app
```


### Create /opt/app folder
Next up we need to create the `/opt/app`-folder, this is where our app will live.  We need to create it so the Github action can scp our tar.gzipped app here and the deploy script can unpack it.

### Add Github workflows
Usually I have 2 workflows in each app, one for running the tests and the other for the actual deploy. I'll show the deploy workflow here. This workflow expects an app that will be deployed using a Wireguard VPN-tunnel.  The workflow has 2 jobs, `build` and `deploy`. The build acts mostly as a cache, if the deploy fails - which happens sometimes, thank you Github - then we can just rerun this step and do not need to do a full recompilation.

```elixir
name: Deploy to production

on:
  push:
    branches:
      - main

env:
  otp-version: 25.0.4
  elixir-version: 1.14.0
  node-version: 17.0.1


jobs:
  build:
    runs-on: ubuntu-22.04

    env:
      MIX_ENV: prod

    steps:
      - uses: actions/checkout@v3

      - name: Install OTP and Elixir
        uses: erlef/setup-beam@v1.14
        with:
          otp-version: ${{ env.otp-version }}
          elixir-version: ${{ env.elixir-version }}

      - name: Cache mix deps
        uses: actions/cache@v3
        env:
          cache-name: mix-deps-prod
        with:
          path: deps
          key: ${{ runner.os }}-${{ env.otp-version }}-${{ env.elixir-version }}-${{ env.cache-name }}-${{ hashFiles(format('{0}{1}', github.workspace, '/mix.lock'))}}

      - name: Cache build
        uses: actions/cache@v3
        env:
          cache-name: mix-build-prod-v1
        with:
          path: _build
          key: ${{ runner.os }}-${{ env.otp-version }}-${{ env.elixir-version }}-${{ env.cache-name }}-${{ hashFiles(format('{0}{1}', github.workspace, '/mix.lock'))}}

      - run: mix deps.get --only prod
      - run: mix deps.compile

  deploy:
    runs-on: ubuntu-22.04
    needs: build

    env:
      MIX_ENV: prod

    steps:
      - uses: actions/checkout@v3

      - name: Install OTP and Elixir
        uses: erlef/setup-beam@v1.14
        with:
          otp-version: ${{ env.otp-version }}
          elixir-version: ${{ env.elixir-version }}

      - name: Cache mix deps
        uses: actions/cache@v3
        env:
          cache-name: mix-deps-prod
        with:
          path: deps
          key: ${{ runner.os }}-${{ env.otp-version }}-${{ env.elixir-version }}-${{ env.cache-name }}-${{ hashFiles(format('{0}{1}', github.workspace, '/mix.lock'))}}

      - name: Cache build
        uses: actions/cache@v3
        env:
          cache-name: mix-build-prod-v1
        with:
          path: _build
          key: ${{ runner.os }}-${{ env.otp-version }}-${{ env.elixir-version }}-${{ env.cache-name }}-${{ hashFiles(format('{0}{1}', github.workspace, '/mix.lock'))}}

      - name: Install Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.node-version }}

      - name: Cache node modules
        uses: actions/cache@v3
        env:
          cache-name: npm-deps-v1
        with:
          path: ~/.npm
          key: ${{ runner.os }}-${{ env.node-version }}-${{ env.cache-name }}-${{ hashFiles(format('{0}{1}', github.workspace, '/assets/package-lock.json'))}}

      - run: |
          npm ci
        working-directory: ./assets

      - run: mix compile
      - run: mix assets.deploy
      - run: mix release --overwrite
      - run: tar -zcf app.tar.gz -C _build/prod/rel/app .

      - name: WireGuard
        uses: Hermanverschooten/wireguard@v0.0.01-alpha
        with:
          config: '${{ secrets.WIREGUARD }}'

      - name: Publish
        uses: nogsantos/scp-deploy@master
        with:
          src: ./app.tar.gz
          host: <host-vpn-ip>
          remote: /opt/app
          port:  22
          user:  root
          key: ${{ secrets.SSH_KEY }}

      - name: Remote SSH Commands
        uses: fifsky/ssh-action@v0.0.4
        with:
          host: <host-vpn-ip>
          port:  22
          user:  root
          key: ${{ secrets.SSH_KEY }}
          command: ./deploy
```
### Add the necessary secrets to Github
If Wireguard is not needed, the step can be removed, the `<host-vpn-ip>` needs to be an IP on which the Github action can access your ssh-server.
If you do want to use Wireguard, supply a valid Wireguard configuration file in your Github secrets.

Your ssh private key must also be present as a Github secret called `SSH_KEY`.

You are free to replace any of these settings with more Github secrets

### Wireguard?
Why do I use Wireguard in this deploy? Most of my ProxMox containers are only available on IPv6 addresses and Github actions currently does not support IPv6. This means I cannot access the server directly. Each of these instances has a secondary network interface that has a private-range-ip. I have setup a Wireguard server that allows access to that range.

### Webserver
Recently I have begun to deploy apps that use `SiteEncrypt` for https, this removes the need to setup a reverse-proxy in front of the app. It brings in a bit more configuration, you can find most of the info on [HexDocs](https://hexdocs.pm/site_encrypt/readme.html).  You'll probably need to add some more `Environment` keys to the unit-file, but that is a topic of another, maybe future, blog-post.

The attentive reader will have noticed that most of my VPS only have an IPv6 address... how are they reachable for IPv4? I use a set of `haproxy` instances for this. Depending on the webserver using either `Proxy-Protocol` or just plain `https`.

<br/>
**UPDATE 2022-12-13**
Due to a Node.js deprecation in the Github Action workers, it was necessary to update the versions of most actions.
