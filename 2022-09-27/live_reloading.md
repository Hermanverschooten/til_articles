visible
-- TITLE --
Triggering page reloads in development
-- TAGS --
elixir
phoenix
-- TLDR --
When I am writing these articles on my dev machine, wouldn't it be nice if the page reloaded on change?
-- CONTENT --
# Triggering page reloads in development

When I am writing these articles on my dev machine, it pains me to have to call the refresh-link, each and everytime I save the file.  Wouldn't it be great if I could do this automatically?

### What to use to watch my files?

There is javascript utility called `chokidar`, but I do not want to add npm-dependencies.
Can I write something in Go or Rust, neither language I am familiar with.
There is something written that you can use from Ruby.

**But wait a minute!** 🤦 _Phoenix_ does this already!

It has a definition within your `config/dev.exs` that tells it to watch certain directories and recompile/reload when one of the files therein changes.

Ok, let's investigate... It's in `phoenix_live_reload`, hmm, hmm, aha, there it is, it uses `FileSystem` to watch for file changes.

### FileSystem

```elixir
{:ok, pid} = FileSystem.start_link(dirs: [article_path])
FileSystem.subscribe(pid)
```
This handles the watching part, and this the listening part.

```elixir
    def handle_info({:file_event, _watcher_pid, {_path, _events}}, state) do
      IO.inspect("File changed, reloading articles")
      {:noreply, state}
    end
```

Now I only want this in development mode, so my `ArticleServer`, which is a `GenServer`, can handle this for me.

```elixir
  if Mix.env() == :dev do
    def init(_) do
      article_path = Application.get_env(:til, :article_path)
      {:ok, pid} = FileSystem.start_link(dirs: [article_path])
      FileSystem.subscribe(pid)

      {:ok, %__MODULE__{}, {:continue, :load_articles}}
    end

    def handle_info({:file_event, _watcher_pid, {_path, _events}}, _state) do
      IO.inspect("File changed, reloading articles")
      {:noreply, load_articles()}
    end
  else
    def init(_) do
      {:ok, %__MODULE__{}, {:continue, :load_articles}}
    end
  end
```

### Triggering the reload

Now for the final part we need to tell our browser through `phoenix_live_reload` that the page should reload.
I started digging through the code, asked on Slack for a _documented_ way to do this, but in the end it was sooooo easy!
The live reloader lives in an `iframe` and listens on a channel, I just needed to check what is being sent on a page reload. 

```elixir
[null,null,"phoenix:live_reload","assets_change",{"asset_type":"heex"}]
```

So let's do the same thing.

```elixir
TilWeb.Endpoint.broadcast("phoenix:live_reload", "assets_change", %{asset_type: "heex"})
```

And it works! And as `@kernel` pointed out, this is the closest I am going to get to a _'supported'_ way, as the channel api is public.

### Full code

So here is the complete code I added:

```elixir
  if Mix.env() == :dev do
    def init(_) do
      article_path = Application.get_env(:til, :article_path)
      {:ok, pid} = FileSystem.start_link(dirs: [article_path])
      FileSystem.subscribe(pid)

      {:ok, %__MODULE__{}, {:continue, :load_articles}}
    end

    def handle_info({:file_event, _watcher_pid, {_path, _events}}, _state) do
      IO.inspect("File changed, reloading articles")
      TilWeb.Endpoint.broadcast("phoenix:live_reload", "assets_change", %{asset_type: "heex"})
      {:noreply, load_articles()}
    end
  else
    def init(_) do
      {:ok, %__MODULE__{}, {:continue, :load_articles}}
    end
  end
```

Writing this article was a pleasure with my page auto-reloading on each save.
And it will also be the first article to be published with my newly added Github Action.
