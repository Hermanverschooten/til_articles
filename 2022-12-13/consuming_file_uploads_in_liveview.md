visible
-- TITLE --
Consuming file uploads in Phoenix live view
-- TAGS --
elixir
phoenix
liveview
upload
-- TLDR --
Phoenix liveview offers a nice way to upload files, or does it?
-- CONTENT --
# Consuming file uploads in Phoenix live view

Very soon I'll be starting a new chapter in my career, after being part of my mother's company since 2005 my department will be split off into it's own company called _Octarion_.
One of the things I needed to do in preparation of this, is creating a site that allows me to keep track of my invoices, bills, payments, etc...

To keep track of my incoming and outgoing payments, my bank provides me with a `CSV` file each day there are new transactions.
So I want to be able to upload these files, process them and see the progress of this in my live view page.

First let's create a form with a live view upload:
![selected](assets/selected.png)
In my `finance_live` page I added a `mount` with

```elixir
@impl true
def mount(_params, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(Admin.PubSub, "finance")
  end

  {:ok,
   socket
   |> assign(messages: %{})
   |> allow_upload(:csv, accept: ~w(.csv), max_entries: 4)}
end
```
This prepares our live view for file uploads, restricts it to files with a `.csv` extension, and limits to 4 files at a time.

```elixir
<form id="upload-form" phx-submit="upload" phx-change="validate">
  <section phx-drop-target={@uploads.csv.ref} class="br-neutral-50">
    <label class="cursor-pointer">
      <%= img_tag(~p"/images/upload_csv.svg") %>
      <.live_file_input upload={@uploads.csv} class="hidden" />
    </label>
    <%= for entry <- @uploads.csv.entries do %>
      <div class="flex items-center">
        <progress value={entry.progress} max="100"><%= entry.progress %> %</progress>
        <span class="px-1"><%= entry.client_name %></span>
        <button
          type="button"
          phx-click="cancel-upload"
          phx-value-ref={entry.ref}
          aria-label="cancel"
        >
          &times;
        </button>
      </div>
      <%= for err <- upload_errors(@uploads.csv, entry) do %>
        <p class="border rounded border-red-600 bg-red-200 text-red-600 p-1">
          <%= error_to_string(err) %>
        </p>
      <% end %>
    <% end %>
  </section>
  <%= for err <- upload_errors(@uploads.csv) do %>
    <p class="border rounded border-red-600 bg-red-200 text-red-600 p-1">
      <%= error_to_string(err) %>
    </p>
  <% end %>
  <button
    type="submit"
    class="border rounded px-4 bg-blue-400 hover:bg-blue-700 text-white mt-1 disabled:bg-gray-100"
    disabled={upload_disabled(@uploads.csv, @messages)}
  >
    Upload
  </button>
</form>
```

That's quite a bit of code, let's dissect it a bit.
We create a form with a `phx-submit` and `phx-change`, nothing out of the ordinary... although... when I look at the code code for the `validate`;
```elixir
@impl true
def handle_event("validate", _params, socket) do
  {:noreply, socket |> assign(messages: %{})}
end

@impl Phoenix.LiveView
def handle_event("cancel-upload", %{"ref" => ref}, socket) do
  {:noreply, cancel_upload(socket, :csv, ref)}
  {:noreply, cancel_upload(socket, :csv, ref)}
end
```
It doesn't really do much, does it. Still it is necessary, or the file uploads will not work.
The second function handles us cancelling the upload for any already added files.

I have chosen to hide the default file selection button and replace it with an image and allow files to be dropped.
Each time I file is selected/dropped the `validate` event will allow live view to record this in `@uploads.csv.entries`, we display each entry with a progress bar and a button to cancel that file's upload.
Any errors are show below the file list. There are 2 types of errors that can occur and Phoenix live view offers us the `upload_errors` function in 2 arities, `/1` returns a general error of `:too_many_files`, the `/2` returns any errors per entry, `:too_large` and `:not_accepted`.

## Consuming files
Phoenix live view uses a in my opinion dubious term of `consuming` files.
To me at first the act of `consuming` a file meant processing it. But to Phoenix it means we have no more use for the file and Phoenix may forget, remove, make it no longer available.
`consume_uploaded_entries/3` takes the `socket`, our field name and a function of arity 2, that will return either a `{:ok, my_result}` meaning we're done with this file, or `{:postpone, my_result}` telling Phoenix to hold on to the file a bit longer, we still need it.
```elixir
@impl Phoenix.LiveView
def handle_event("upload", _params, socket) do
  entries =
    consume_uploaded_entries(socket, :csv, fn %{path: path}, entry ->
      {:postpone, {path, entry}}
    end)

  process_entries(self(), entries)

  {:noreply, socket}
end

defp process_entries(pid, entries) do
  tasks =
    for {path, entry} <- entries do
      Task.async(fn -> process(pid, path, entry) end)
    end

  Task.await_many(tasks)
  Phoenix.PubSub.broadcast(Admin.PubSub, "finance", :refresh)
end

defp process(pid, path, entry) do
  Process.send(pid, {:add_message, entry, "Starting"}, [:noconnect])

  case Finance.parse(entry.client_name, path) do
    {:ok, []} ->
      Process.send(pid, {:add_message, entry, "No content"}, [:noconnect])

    {:ok, [first | rest]} ->
      Process.send(pid, {:add_message, entry, "Parsed content"}, [:noconnect])

      case Finance.fetch_account(first.account_number) do
        {:ok, account} ->
          Process.send(pid, {:add_message, entry, "Found account #{account.account}"}, [
            :noconnect
          ])

          for trx <- [first | rest] do
            case Finance.add_transaction(account.id, trx) do
              {:ok, _btrx} ->
                Process.send(
                  pid,
                  {:add_message, entry, "#{trx.number}/#{trx.line} #{trx.amount} #{trx.party}"},
                  [
                    :noconnect
                  ]
                )

              {:error, changeset} ->
                Process.send(
                  pid,
                  {:add_message, entry,
                   "#{trx.number}/#{trx.line} failed. #{inspect(changeset.errors)}"},
                  [
                    :noconnect
                  ]
                )
            end
          end

        {:error, :not_found} ->
          Process.send(
            pid,
            {:add_message, entry, "Unable to find a record for '#{first.account_number}'"},
            [:noconnect]
          )
      end

    {:error, :unknown_account} ->
      Process.send(pid, {:add_message, entry, "Unkown bank account"}, [:noconnect])
  end

  Process.send(pid, {:finished, entry}, [:noconnect])
end

defp error_to_string(:too_large), do: "Too large"
defp error_to_string(:too_many_files), do: "You have selected too many files"
defp error_to_string(:not_accepted), do: "You have selected an unacceptable file type"

defp upload_disabled(%{entries: []}, _messages), do: true
defp upload_disabled(_entries, messages) when messages != %{}, do: true
defp upload_disabled(%{errors: []}, _messages), do: false
defp upload_disabled(_, _), do: true

@impl true
def handle_info({:add_message, %{client_name: file}, msg}, socket) do
  messages = Map.get(socket.assigns.messages, file, [])

  file_messages = Map.put(socket.assigns.messages, file, messages ++ [msg])
  {:noreply, socket |> assign(messages: file_messages)}
end

@impl true
def handle_info({:finished, entry}, socket) do
  consume_uploaded_entry(socket, entry, fn _ -> {:ok, ""} end)
  {:noreply, socket}
end
```

The `upload` event creates a list of files to process, which is then given to `process_entries/2`, this creates a `Task` for each file and _awaits_ the completion of the processing, after which the info on the page is refreshed (function not shown).
The `process/3` function takes the `pid` of the live view process, the path to the uploade file and the live view file upload entry. It starts by sending a message to the process that we are starting.
This message is added to the page by the `handle_info/2`. It then continues processing the transactions in the file, sending adding a message to the page each time.
![processed](assets/processed.png)
Once a file has been processed a `{:finished, entry}` message is send to the live view process, this then uses the `consume_uploaded_entry/3` function to tell Phoenix live view to release the file.

By using `Task` and `Process.send` in this way allows us to communicate with the live view page.

The rest of the code are some helper functions to display errors and disable the `upload button` when necessary.

