visible
-- TITLE --
Transform from one Ecto.Schema to another with one command
-- TAGS --
elixir
ecto
-- TLDR --
You can use `Schema.load/2` to transform from 1 `Ecto.Schema` to another.
-- CONTENT --
# Transform from one Ecto.Schema to another with one command
Recently I was struggling with a polymorphic table I made.
The table contains documents of 4 different types, but they all adhere to almost the same structure.
My urgent issue was that each of these types have a different embedded schema.
I looked and tried and found a hack to load the schema.

```elixir
def to_data(%{type: type, data: data} = document) do
  module =
    Module.concat(Schema, String.capitalize(to_string(type)))
    |> Module.concat(Data)

  Map.put(
    document,
    :data,
    struct(module)
    |> Ecto.Changeset.cast(data, module.__schema__(:fields))
    |> Ecto.Changeset.apply_changes()
  )
end
```

But it turns out `Ecto` has a way to make this much easier - `Ecto.load/2`.

```elixir
doc = Repo.get(Schema.Document, 1)
Ecto.load Schema.Invoice, doc
```

This returns a `%Schema.Invoice{}` with the data from `doc`, including the embedded schemas.

### Beware!

This will not work for 'converted' values such as `Ecto.Enum`, it needs the raw/database value.

Neither will it copy loaded associations.

But still another useful tool for my belt.

