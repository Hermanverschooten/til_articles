visible
-- TITLE --
Merging PDF's 
-- TAGS --
pdf
elixir
phoenix
-- TLDR --
A customer wanted to send a generated PDF merged with a number of existing PDF's by e-mail.
-- CONTENT --
# Merging PDF's
One of my customers wants to send a merged PDF document by e-mail from within their Phoenix application.
The first document is a generated PDF, the other documents are PDF's created by a scanner.

To generate the PDF we use the `pdf` package from [ hex.pm ](https://hex.pm/packages/pdf).

A basic PDF can be generated as easily as
```elixir
Pdf.build([size: :a4, compress: true], fn pdf ->
  pdf
  |> Pdf.set_info(title: "Demo PDF")
  |> Pdf.set_font("Helvetica", 10)
  |> Pdf.text_at({200,200}, "Welcome to Pdf")
  |> Pdf.write_to("test.pdf")
end)
```
The other PDF's are all stored in a folder that is accessible by the app.

## The interface
I use a live_view with a number of `async` assigns to show the process interactively.
![screenshot](assets/screenshot.png)

In the live_view `mount/3` function we do a couple of assigns.
```elixir
 {
   :ok,
   socket
   |> assign(emails: emails)
   |> assign(:document, document)
   |> assign(:filename, nil)
   |> assign(:invoice_pdf, AsyncResult.loading())
   |> assign(:documents, AsyncResult.loading())
   |> assign(:merge, AsyncResult.loading())
   |> assign(:mail, AsyncResult.loading())
 }
```
And use them to show the different states.
```elixir
<div class="grid grid-cols-2 border rounded p-4 my-4">
  <div class="border-b pb-2">Opzoeken e-mail bestemming(en)</div>
  <div class="border-b pb-2">
    <%= if @emails == [] do %>
      <.icon name="hero-x-mark" class="w-4 h-4 text-red-500" />
    <% else %>
      <button phx-click="start" class="px-4 border rounded bg-gray-100 -ml-4">
        <.icon name="hero-check" class="w-4 h-4 text-green-500" /> Verzend
        <span class="ml-1 text-sm text-gray-400"><%= Enum.join(@emails, ", ") %></span>
      </button>
    <% end %>
  </div>
  <div class="border-b py-2">Aanmaken factuur PDF</div>
  <div class="border-b py-2">
    <.async_result assign={@invoice_pdf}>
      <:loading>
        <.icon name="hero-clock" class="w-4 h-4 text-gray-500" />
      </:loading>
      <:failed :let={_reason}>
        <.icon name="hero-x-mark" class="w-4 h-4 text-red-500" />
      </:failed>
      <div>
        <.icon name="hero-check" class="w-4 h-4 text-green-500" />
      </div>
    </.async_result>
  </div>
  <div class="border-b py-2">Verzamelen vracht documenten</div>
  <div class="border-b py-2">
    <.async_result assign={@documents}>
      <:loading>
        <.icon name="hero-clock" class="w-4 h-4 text-gray-500" />
      </:loading>
      <:failed :let={_reason}>
        <.icon name="hero-x-mark" class="w-4 h-4 text-red-500" />
      </:failed>
      <div>
        <.icon name="hero-check" class="w-4 h-4 text-green-500" />
      </div>
    </.async_result>
  </div>
  ...
</div>

```

When the client presses the send button, we kick of the process.
```elixir
  def handle_event("start", _, socket) do
    id = socket.assigns.id

    {:noreply,
     socket
     |> start_async(:generate_invoice_pdf, fn ->
       generate_pdf(id)
     end)}
  end
  def handle_async(:generate_pdf, {:ok, pdf}, socket) do
    %{invoice_pdf: invoice_pdf} = socket.assigns

    {:noreply,
     socket
     |> assign(:invoice_pdf, AsyncResult.ok(invoice_pdf, pdf))
     |> start_async(:documents, fn -> gather_documents(socket.assigns.document) end)}
  end
```
Once the PDF is generated the `handle_async/2` callback is called, we assign the document to the `AsyncResult` which updates our display and start the next task.
The `gather_documents` task looks up the documents in the database and returns a list of paths, once done returns and we kick of the `:merge` task.
This is the most interesting part. I first looked at [qpdf](https://qpdf.sourceforge.io/) to do the merging and this worked flawlessly, but the size of the merged pdf quicky grew too large.
Especially as the scanned documents have a way to high resolution, trying to use the optimization only yielded 36k on a 5.81M file. Not enough.
The qpdf commandline I tried with was: `qpdf --recompress-flate --compression-level=9 --object-streams=generate --empty --pages invoice.pdf extra.pdf extra2.pdf -- output.pdf`
Looking for a way to compress more I stumbled on [ghostscript](https://www.ghostscript.com/), which cli utility `gs` seems to be available both on my Mac and my Ubuntu server.
```elixir
  defp merge(invoice_path, document_list) do
    output_filename = Ecto.UUID.generate() <> ".pdf"
    output_path = Path.join(Application.get_env(:app, :tmp_path, "/tmp"), output_filename)

    args =
      [
        "-sDEVICE=pdfwrite",
        "-dCompatibilityLevel=1.4",
        "-dPDFSETTINGS=/ebook",
        "-dNOPAUSE",
        "-dQUIET",
        "-dBATCH",
        "-sOutputFile=#{output_path}",
        invoice_path | document_list
      ]

    case Porcelain.exec("gs", args, out: :string, err: :out) do
      %{status: 0} ->
        File.rm_rf(invoice_path)
        size = File.stat!(output_path).size

        {:ok, %{filename: output_filename, path: output_path, size: size}}

      e ->
        {:error, e}
    end
  end

```
The information is again returned to the `handle_async/2` callback, and attached to the mail and once sent, I clean up the generated file. 

Using the new async possibilities of Phoenix makes this a breeze.

![movie](assets/example.gif)
The animation shows the entire flow in action. I halt just before sending to give them the opportunity to check the PDF and it's size before continuing.
The only downside is if they don't like it they have to press the _NEE_ button else the file will remain in the `/tmp` folder.
I may make a recurring task to do the clean up anyway.
