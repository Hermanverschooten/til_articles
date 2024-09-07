visible
-- TITLE --
Adding Passkey support to Phoenix
-- TAGS --
passkey
Webauthn
authentication
elixir
phoenix
-- TLDR --
It was time to add Passkey support to my Phoenix app.
-- CONTENT --
# Adding Passkey support
I am not going to explain what Passkeys are, there are sufficient [resources](https://hexdocs.pm/webauthn_components/readme.html#additional-resources) on the web already.
I have a Phoenix-based website for my invoicing and crm-related tasks.
It currently uses a login-form that posts to a `session_controller`.
It works fine, but the Passkey buzz has me interested in adding support for it.
I read about  the [`WebAuthnComponent`](https://hexdocs.pm/webauthn_components/readme.html) package by [Owen bickford](https://elixirforum.com/u/type1fool/summary).

The package has a nice readme, and explains how to add passkey support to a new Phoenix application... but that is not what I want.
I spent a couple of hours reading the code and the accompanying demo application. I got the drift, but the `UserToken` took me some extra time to grok.

## So how to add this to my current app?

We start out with adding the package to our mix file and running `mix deps.get`
```elixir
     {:webauthn_components, "~> 0.7"}
```

A large part of the authentication flow is handled with Javascript, so let's first add these parts to our `app.js`
```elixir

import {
  SupportHook,
  AuthenticationHook,
  RegistrationHook,
} from "webauthn_components";

let Hooks = { SupportHook, AuthenticationHook, RegistrationHook }
```
I already had a couple of hooks, so initializing my `Hooks` variable with the new hooks was straightforward.
That's all for the Javascript part, nice.

Now, I need a place where I can add my Passkey. Oh wait, I need to create a table for the keys and make some changes to my user.
Contrary to most I do not nest my schemas below my contexts, all my schemas live under `Schema`.

```elixir
mix ecto.gen.migration create_user_keys

defmodule MyApp.Repo.Migrations.CreateUserKeys do
  use Ecto.Migration

  def change do
    create table(:user_keys, primary_key: false) do
      add :id, :binary_id, primary_key: true
      add :user_id, references(:users, on_delete: :delete_all), null: false
      add :label, :string, null: false
      add :key_id, :binary, null: false
      add :public_key, :binary, null: false
      add :last_used_at, :utc_datetime, null: false, default: fragment("now()")

      timestamps()
    end

    create index(:user_keys, [:user_id])
    create unique_index(:user_keys, [:key_id])
    create unique_index(:user_keys, [:user_id, :label])
  end
end

defmodule Schema.UserKey do
  @moduledoc """
  Schema representing a `User`'s Passkey / Webauthn credential.

  ## Considerations

  - A user may have multiple keys.
  - Each key must have a unique label.
  - `:last_used_at` is set when the key is created and updated, and this value cannot be cast through the changesets.
  """
  use Ecto.Schema
  import Ecto.Changeset
  alias WebauthnComponents.CoseKey

  @type t :: %__MODULE__{
          id: binary(),
          label: String.t(),
          key_id: binary(),
          public_key: map(),
          last_used_at: NaiveDateTime.t(),
          inserted_at: NaiveDateTime.t(),
          updated_at: NaiveDateTime.t()
        }

  @primary_key {:id, Ecto.ULID, autogenerate: true}
  @derive {Jason.Encoder, only: [:key_id, :public_key, :label, :last_used_at]}
  schema "user_keys" do
    field :label, :string, default: "default"
    field :key_id, :binary
    field :public_key, CoseKey
    belongs_to :user, Schema.User
    field :last_used_at, :naive_datetime

    timestamps()
  end
end
```

The above schema was copied almost literally from the demo app, I just removed all the `changeset` functions, as I prefer to keep them where needed.
Now add a reference in `Schema.User`, again like in the demo app.

```elixir
 :keys, Schema.UserKey, preload_order: [desc: :last_used_at]
```

Generate the table with `mix ecto.migrate`, and we are ready for the next part.

### Registration

I recently added a `Profile` page, so that seems to be the best place to add the Passkey registration.
We add some aliases for convenience.

```elixir
  alias WebauthnComponents.RegistrationComponent
  alias WebauthnComponents.SupportComponent
  alias WebauthnComponents.WebauthnUser
```

We set a `passkey` assign to `false` to begin with, this will tell us if passkey authentication is supported in this browser. 

```elixir
    {:ok,
     socket
     ...
     |> assign(passkey: false)}
```

Then in our `render/1` function we add 2 live components.

```elixir
  def render(assigns) do
    ~H"""
    <.live_component module={SupportComponent} id="support-component" />
    <h1 class="mt-3 text-xl font-semibold">Profile</h1>
    <.simple_form for={@form} phx-change="validate" phx-submit="save">
    ...
      <:actions>
        <.button>Save</.button>
        <div :if={@passkey}>
          <.live_component
            module={RegistrationComponent}
            id="registration-component"
            app={MyApp}
            display_text="Passkey"
            class={[
              "bg-gray-200 hover:bg-gray-300 hover:text-black rounded transition",
              "ring ring-transparent focus:ring-gray-400 focus:outline-none",
              "flex gap-2 items-center justify-center px-4 py-2 w-full",
              "disabled:cursor-not-allowed disabled:opacity-25"
            ]}
          />
        </div>
      </:actions>
    </.simple_form>
    """
  end
```
The `SupportComponent` checks if Passkey support is available, which we need to handle.
```elixir
  def handle_info({:passkeys_supported, true}, socket) do
    current_user = socket.assigns.current_user

    webauthn_user = %WebauthnUser{
      id: generate_encoded_id(),
      name: current_user.login,
      display_name: current_user.firstname <> " " <> current_user.name
    }

    send_update(RegistrationComponent, id: "registration-component", webauthn_user: webauthn_user)

    {:noreply,
     socket
     |> assign(webauthn_user: webauthn_user)
     |> assign(passkey: true)}
  end

  def handle_info({:passkeys_supported, false}, socket) do
    {:noreply,
     socket
     |> assign(passkey: false)}
  end
  ...
  defp generate_encoded_id do
    :crypto.strong_rand_bytes(64)
    |> Base.encode64(padding: false)
  end
```
If available we initialize the `WebauthnUser` with the current user's information and send this to the `RegistrationComponent`.
The `RegistrationComponent` renders a button that starts the registration process, but all we need to handle is...

```elixir

  def handle_info({:registration_successful, params}, socket) do
    with {:ok, _key} <- MyApp.User.add_passkey(socket.assigns.current_user, params[:key]) do
      {:noreply, socket |> put_flash(:info, "Passkey added")}
    else
      {:error, changeset} ->
        {:noreply,
         socket |> put_flash(:error, "Failed to add passkey, #{inspect(changeset.errors)}")}
    end
  end

  def handle_info(msg, socket) do
    {:noreply, socket |> put_flash(:error, inspect(msg))}
  end
```
The `add_passkey/2` function inserts the key into the `user_keys` table.

And that completes the registration part. Phew!

### Authentication

For my authentication I decided to create a new live_view page.

```elixir
defmodule MyAppWeb.LoginLive do
  use MyAppWeb, :live_view
  alias WebauthnComponents.AuthenticationComponent
  alias WebauthnComponents.SupportComponent

  def mount(_params, _session, socket) do
    form = %{"login" => "", "password" => ""} |> to_form()

    {:ok,
     socket
     |> assign(form: form)
     |> assign(passkey: false)}
  end

  def render(assigns) do
    ~H"""
    <.live_component module={SupportComponent} id="support-component" />
    <.flash_group flash={@flash} />
    <div class="w-screen h-screen flex justify-center items-center">
      <div class="p-4 border rounded bg-[#cccccc] w-full sm:w-1/2 xl:w-1/3 flex flex-col sm:flex-row items-center gap-2">
        <img src={~p"/images/logo.png"} class="self-center w-32 md:mr-4 md:pr-3 bg-transparent" />
        <.simple_form
          :let={f}
          for={@form}
          phx-change="validate"
          phx-submit="login"
          class="flex flex-col gap-2 flex-1 md:border-l md:pl-4 min-w-min bg-transparent"
        >
          <input
            type="text"
            name="login"
            value={f[:login].value}
            placeholder="login"
            class="rounded w-full mb-2"
          />
          <input type="password" name="password" placeholder="password" class="rounded w-full" />
          <:actions>
            <.button>Logon</.button>
            <.live_component
              :if={@passkey}
              module={AuthenticationComponent}
              id="authentication-component"
              display_text="Passkey"
              class={[
                "bg-gray-200 hover:bg-gray-300 hover:text-black rounded transition",
                "ring ring-transparent focus:ring-gray-400 focus:outline-none",
                "flex gap-2 items-center justify-center px-4 py-2 w-full",
                "disabled:cursor-not-allowed disabled:opacity-25"
              ]}
            />
          </:actions>
        </.simple_form>
      </div>
    </div>
    """
  end

  def handle_event("validate", params, socket) do
    {:noreply, socket |> assign(form: to_form(params))}
  end

  def handle_event("login", %{"login" => login, "password" => password}, socket) do
    with {:ok, user} <- MyApp.Authentication.authenticate(login, password) do
      redirect_to_session(user, socket)
    else
      {:error, _} ->
        {:noreply,
         socket
         |> put_flash(:error, "There was an issue with your credentials!")}
    end
  end

  def handle_info({:passkeys_supported, supported?}, socket) do
    {:noreply,
     socket
     |> assign(passkey: supported?)}
  end

  def handle_info({:find_credential, [key_id: key_id]}, socket) do
    case MyApp.User.get_by_key_id(key_id) do
      {:ok, user} ->
        send_update(AuthenticationComponent, id: "authentication-component", user_keys: user.keys)

        {
          :noreply,
          socket
          |> assign(:user, user)
          |> assign(:key_id, key_id)
        }

      {:error, :not_found} ->
        {
          :noreply,
          socket
          |> put_flash(:error, "Failed to sign in")
        }
    end
  end

  def handle_info({:authentication_successful, _auth_data}, socket) do
    %{user: user, key_id: key_id} = socket.assigns

    MyApp.User.touch_key(key_id)

    redirect_to_session(user, socket)
  end

  def handle_info({:error, %{"name" => "AbortError"}}, socket) do
    {:noreply,
     socket
     |> put_flash(:warning, "Passkey aborted, please login with credentials")}
  end

  def handle_info({:error, _}, socket) do
    {:noreply,
     socket
     |> put_flash(:error, "Passkey failed, please login with credentials")}
  end

  defp redirect_to_session(user, socket) do
    token =
      Phoenix.Token.sign(MyAppWeb.Endpoint, "Some very secret salt", %{
        user_id: user.id
      })

    {:noreply,
     socket
     |> redirect(to: "/session/#{token}")}
  end
end
```
As before we add the `SupportComponent` and the necessary `handle_info/3` callbacks to know if passkey support is available.
The `AuthenticationComponent` adds a button, but I haven't needed it so far, my browser (Firefox Developper Edition) immediately prompts me to use the Passkey.
When we have selected the correct passkey, the `handle_info/3` callback is called with `:find_credentials` and the supplied `:key_id`, these we need to get the user from the database.
When found we update the component, and assign both the found user and key_id to the socket.
Upon successful authentication, we touch the key to update it's `:last_updated_at` field and redirect to our `SessionController`.
We still need this controller to put our `:user_id` in the session, this is not possible from live_view.
To do this, we protect the `:user_id` with a `Phoenix.Token` that we put in the url.
When we decide not to use the passkey for authentication the `AuthenticationComponent` sends an `:error` tuple with `AbortError`.

If we use the normal password authentication, the same session redirect is used.

In the `SessionController` the `create/2` function is used to complete the flow.
```elixir

  def create(conn, %{"token" => token}) do
    with {:ok, %{user_id: user_id}} <-
           Phoenix.Token.verify(MyAppWeb.Endpoint, "Some very secret salt", token, max_age: 5),
         {:ok, user} <- MyApp.User.fetch(user_id) do
      Sentry.Context.set_user_context(%{id: user.id, username: user.name, email: user.email})

      conn
      |> put_session(:user_id, user.id)
      |> redirect(to: ~p"/")
    end
    else
      _ ->
        conn
        |> put_flash(:error, "There was an issue with your credentials!")
        |> redirect(to: ~p"/login")
    end
  end
```
We fetch the `:user_id` from the `token`, fetch the user, update the session and go to the dashboard.
In case something goes wrong, it's back to the login page.

### Conclusion
At first I thought it would be much more difficult to add Passkey support to my existing Phoenix app, but it turned out okay.
Some things I glossed over are the context functions, the `:on_mount` callback that checks for the `:user_id` and adds the user to the assigns,...
these should not be to difficult.
I do recommend to check out the [demo](https://github.com/liveshowy/webauthn_components_demo) repo, as it contains some extra functionality concerning `UserToken`s, but I decided not to use this for my application.

### Caveat !!!
When I was initially testing the registration it would fail every time with either an `:origin` error or some message about `relying party`, or `the operation is insecure`, it took me some serious time to find the reason.
The package uses [`Wax`](https://hexdocs.pm/wax_) for the _serious_ stuff and in one of the calls `:origin` is set to `endpoint.url()`
This `url/0` function in your Phoenix `Endpoint`, contains in `:dev` something like `http://localhost:4000`.
The `Wax` code checks for `localhost` and changes some settings to allow it.
But in my dev setup I use urls with a `.test` extension over `https`, this was causing the `url` to contain `https://localhost:4443` which is not valid for passkey authentication.
The `:origin` is not allowed to have a port attached to it. The solution was to add `url: [host: "my_app.test", port: 443]` to my endpoint in `config/dev.exs`.

For more information about the library, check out the [post](https://elixirforum.com/t/webauthnlivecomponent-passwordless-auth-for-liveview-apps/49941) on the Elixir Forum.
