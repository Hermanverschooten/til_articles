visible
-- TITLE --
Moving away from Phoenix views
-- TAGS --
elixir
phoenix
-- TLDR --
Today I decided to upgrade one of my projects away from Phoenix views.
These are the steps I took, problems I ran into
-- CONTENT --
# Moving away from Phoenix views

Today I decided to upgrade one of my projects away from Phoenix views.

These are the changes I made:

### Update config/config.exs and add new modules
```elixir
-  render_errors: [view: CraftsThingsWeb.ErrorView, accepts: ~w(html json), layout: false],
+  render_errors: [
+    formats: [html: CraftsThingsWeb.ErrorHTML, json: CraftsThingsWeb.ErrorJSON],
+    layout: false
+  ],

new file mode 100644
index 0000000..9ecd8d8
--- /dev/null
+++ b/lib/crafts_things_web/components/core_components.ex
@@ -0,0 +1,4 @@
+defmodule CraftsThingsWeb.CoreComponents do
+ use Phoenix.Component
+
+end

new file mode 100644
index 0000000..3ed2dcd
--- /dev/null
+++ b/lib/crafts_things_web/controllers/error_html.ex
@@ -0,0 +1,19 @@
+defmodule CraftsThingsWeb.ErrorHTML do
+  use CraftsThingsWeb, :html
+
+  # If you want to customize your error pages,
+  # uncomment the embed_templates/1 call below
+  # and add pages to the error directory:
+  #
+  #   * lib/phx_web/controllers/error_html/404.html.heex
+  #   * lib/phx_web/controllers/error_html/500.html.heex
+  #
+  # embed_templates "error_html/*"
+
+  # The default is to render a plain text page based on
+  # the template name. For example, "404.html" becomes
+  # "Not Found".
+  def render(template, _assigns) do
+    Phoenix.Controller.status_message_from_template(template)
+  end
+end

new file mode 100644
index 0000000..61cc4d2
--- /dev/null
+++ b/lib/crafts_things_web/controllers/error_json.ex
@@ -0,0 +1,15 @@
+defmodule CraftsThingsWeb.ErrorJSON do
+  # If you want to customize a particular status code,
+  # you may add your own clauses, such as:
+  #
+  # def render("500.json", _assigns) do
+  #   %{errors: %{detail: "Internal Server Error"}}
+  # end
+
+  # By default, Phoenix returns the status message from
+  # the template name. For example, "404.json" becomes
+  # "Not Found".
+  def render(template, _assigns) do
+    %{errors: %{detail: Phoenix.Controller.status_message_from_template(template)}}
+  end
+end

new file mode 100644
index 0000000..d528c2e
--- /dev/null
+++ b/test/crafts_things_web/controllers/error_html_test.exs
@@ -0,0 +1,14 @@
+defmodule CraftsThingsWeb.ErrorHTMLTest do
+  use CraftsThingsWeb.ConnCase, async: true
+
+  # Bring render_to_string/4 for testing custom views
+  import Phoenix.Template
+
+  test "renders 404.html" do
+    assert render_to_string(CraftsThingsWeb.ErrorHTML, "404", "html", []) == "Not Found"
+  end
+
+  test "renders 500.html" do
+    assert render_to_string(CraftsThingsWeb.ErrorHTML, "500", "html", []) == "Internal Server Error"
+  end
+end

new file mode 100644
index 0000000..25c6d9d
--- /dev/null
+++ b/test/crafts_things_web/controllers/error_json_test.exs
@@ -0,0 +1,12 @@
+defmodule CraftsThingsWeb.ErrorJSONTest do
+  use CraftsThingsWeb.ConnCase, async: true
+
+  test "renders 404" do
+    assert CraftsThingsWeb.ErrorJSON.render("404.json", %{}) == %{errors: %{detail: "Not Found"}}
+  end
+
+  test "renders 500" do
+    assert CraftsThingsWeb.ErrorJSON.render("500.json", %{}) ==
+             %{errors: %{detail: "Internal Server Error"}}
+  end
+end
```
I opted to add an empty `CoreComponents` for now, instead of copying the proposed one from a newly generated project, so I can see what I need in there.
I then copied my layouts to a new `layouts_html` folder inside the `components` folder, making sure to replace `Routes.static_path` with the correct `~p`-paths.
For now, I added a `root.html.heex` with just `<%= @inner_content %>`, because my other layouts are complete HTML-pages.
Next I added the line `plug :put_root_layout, {CraftsThingsWeb.Layouts, :root}` in my `router.ex`, just above the `plug :put_secure_browser_headers`.

### Update config/dev.exs
```elixir
-      ~r"lib/crafts_things_web/(live|views)/.*(ex)$",
+      ~r"lib/crafts_things_web/(controllers|live|components)/.*(ex)$",
```


### Update lib/crafts_things/crafts_things_web.ex

```elixir
   This can be used in your application as:

       use CraftsThingsWeb, :controller
-      use CraftsThingsWeb, :view
+      use CraftsThingsWeb, :html

-  The definitions below will be executed for every view,
-  controller, etc, so keep them short and clean, focused
+  The definitions below will be executed for every controller,
+  component, etc, so keep them short and clean, focused
   on imports, uses and aliases.

   Do NOT define functions inside the quoted expressions
-  below. Instead, define any helper function in modules
-  and import those modules here.
+  below. Instead, define additional modules and import
+  those modules here.
   """

+  def static_paths(), do: ~w(assets css fonts images js favicon.ico robots.txt)
+
   def controller do
+    quote do
+      use Phoenix.Controller,
+        formats: [:html, :json],
+        layouts: [html: CraftsThingsWeb.Layouts]
+
+      import Plug.Conn
+      import CraftsThingsWeb.Gettext
+
+      unquote(verified_routes())
+    end
+  end
+
+  def old_controller do
     quote do
       use Phoenix.Controller, namespace: CraftsThingsWeb
@@ -58,6 +60,45 @@ defmodule CraftsThingsWeb do
     end
   end

+  def html do
+    quote do
+      use Phoenix.Component
+
+      # Import convenience functions from controllers
+      import Phoenix.Controller,
+        only: [get_csrf_token: 0, view_module: 1, view_template: 1]
+
+      # Include general helpers for rendering HTML
+      unquote(html_helpers())
+    end
+  end
+
+  defp html_helpers do
+    quote do
+      # HTML escaping functionality
+      import Phoenix.HTML
+      # Core UI components and translation
+      import CraftsThingsWeb.CoreComponents
+      import CraftsThingsWeb.Gettext
+
+      # Shortcut for generating JS commands
+      alias Phoenix.LiveView.JS
+
+      # Routes generation with the ~p sigil
+      unquote(verified_routes())
+    end
+  end
+
+  def verified_routes do
+    quote do
+      use Phoenix.VerifiedRoutes,
+        endpoint: CraftsThingsWeb.Endpoint,
+        router: CraftsThingsWeb.Router,
+        statics: CraftsThingsWeb.static_paths()
+    end
+  end
+
```

I renamed the existing `controller` method to `old_controller` and changed it in all existing controllers.
This prepares the way for switching over to the new style.

### Converting the first controller

Let's see how far we get, change the `:old_controller` do `:controller` and refresh the page.
This will ofcourse gives an error as we have not yet defined a `_html` module. So let's create that now.
```elixir
defmodule CraftsThingsWeb.PageHTML do
  use CraftsThingsWeb,: html

  embed_templates "page_html/*.html"
end
```

and move over the html-files from the templates folder to a folder called `page_html` underneath `controllers`, in
the mean time renaming `.html.eex` to `.html.heex` to enjoy the new html-checking, ...

Hmm, that is no fun!

I have a category menu that is renderen in each file on the top of the page, using `render/2` which no longer exists.
So we need to convert this `_category.html.heex` into a component, not that difficult. Create a `category.ex` underneath `components`.
```elixir
def CraftsThingsWeb.Category do
  use Phoenix.Component

  def categories(assigns) do
  ~H"""
  past the html here
  """
  end

  copy any function needed in the above html from `page_view`.
end
```
Now change the `render` into `<CraftsThingsWeb.Category.categories categories={@categories} conn={@conn} />`

I need to pass in `@conn`, because I need to access the sessions, and I have not yet found how to do it without.

Next up, then 'link/2' function does not exist anymore, at least it is not availabel from `Phoenix.Component`, so replace them all with the new `.link`-component.
Then `img_tag/2`, I decided to replace them with regular HTML `img` tags.

While doing this I also needed to replace all references to `Routes.page_path/2` to the new verified routes `sigil_p` syntax.

A `form_for`, what should I do with this one. Well I have no changeset and I am not using the `f` variable, so let's replace it with a regular HTML `form`,
and do not forget to add a hidden `input` with the csrf value.

Those were all the changes I needed to make. 
That was not so bad.

I went and deleted the `page_view` module, but that was too soon, it is referenced from another page... so undo the delete, and time to commit.

### Formatting
Oh before I forget, since we are using `heex`-files now, we need to make some changes to the `.formatter.exs` file.
```elixir
-  import_deps: [:ecto, :phoenix],
-  inputs: ["*.{ex,exs}", "priv/*/seeds.exs", "{config,lib,test}/**/*.{ex,exs}"],
+  import_deps: [:ecto, :ecto_sql, :phoenix],
+  plugins: [Phoenix.LiveView.HTMLFormatter],
+  inputs: ["*.{heex,ex,exs}", "priv/*/seeds.exs", "{config,lib,test}/**/*.{heex,ex,exs}"],
```

I must say that I like the changes I made so far, I prefer the new location of the HTML-files,
and love getting away from the `Phoenix.HTML.Tags` functions and use the more HTML-like components, or just plain old HTML.

### Mini-components?
Another thing I learned was that in `pageHTML` you cannot only use functions but also component-like functions.
```elixir
  def prefix(assigns) do
    ~H"""
    <%= if @product.category.menu == "S" do %>
      <%= render_slot(@inner_block) %> <%= @product.category.name %>
    <% else %>
      <%= @product.category.name %>
    <% end %>
    """
  end
```

and then just use it like
```elixir
<.prefix product={@product}>
    <span class="text-gray-500">stempels</span>
</.prefix>
```

Now let's continue with the other controllers and see what they have in store for us...

