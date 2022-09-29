visible
-- TITLE --
Upgrading to Phoenix LiveView 0.18
-- TAGS --
elixir
phoenix
live_view
-- TLDR --
Today I decided to upgrade one of my more recent projects to live_view 0.18.
These are the steps I took, problems I ran into
-- CONTENT --
# Upgrading an existing project to LiveView 0.18

Today I decided to upgrade one of my more recent projects to live_view 0.18.
After making the necessary changes to my `mix.exs` file and `mix deps.get`, I immediately ran into an issue.

```elixir
== Compilation error in file lib/admin_web/views/layout_view.ex ==
** (CompileError) lib/admin_web/templates/layout/live.html.heex:1: undefined function live_component/1
(expected AdminWeb.LayoutView to define such a function or for it to be imported, but none are available)
```

I am using a `live_component` in my layout-view, I am already using the new `<.live_component module={...} id={...}>` syntax.
I took a peek at my go-to project [LiveBeats](https://github.com/fly-apps/live_beats), noticed they had added a `use Phoenix.Component` to their `LiveBeatsWeb.LayoutView`.

Added it too, and yeah compilation! 

`@jonr` on Slack suggested that I instead add an `import Phoenix.Component` in my `admin_web.ex` macro, as mentioned in the [docs](https://hexdocs.pm/phoenix_live_view/installation.html#existing-projects).

On to the next issue.

```elixir
function Phoenix.LiveView.assign/2 is undefined or private
```

What the? Huh? 

I have an `on_mount` module in my router, that checks for the current logged in user and assigns it to the socket.
Apparently this too has been moved to `Phoenix.Component`.

These 2 seem to have been the most immediate issues to solve, the app functions.

Now let's see if there are any future deprecations we can solve.

## LiveHelpers deprecation
In the [Changelog](https://hexdocs.pm/phoenix_live_view/changelog.html#0-18-0-2022-09-20) is mentioned that a number of helpers have been replaced by new function components, `live_redirect`, `live_patch`, `push_redirect`.

So I decided to remove the import from my 'app_web.ex', to see where it would explode.
These are the ones I need to change:
 - `live_title_tag`, replace with `<.live_title>`
 - `live_file_input`, replace with `<.live_file_input upload={}>`
 - `live_redirect`, replace with `<.link navigate={}>`

 So I was happily replacing tags when all of a sudden, **BOOM**

 I replaced `<%= link to: "mailto:#{customer.email}" do %>..<% end %>` with `<.link href={"emailto:#{customer.email}}>..</.link>`, that however caused 

 ```elixir
** (exit) an exception was raised:
    ** (ArgumentError) unsupported scheme given to <.link>. In case you want to link to an
unknown or unsafe scheme, such as javascript, use a tuple: {:javascript, rest}
 ```

 Damn, it is so clever to recognize I had misspelled **'mailto'**!


