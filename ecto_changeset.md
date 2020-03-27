### Cast assoc

`cast_assoc/3` works matching the records extracted from the database (preload) and compares it with the parameters provided from an external source.

```elixir
cast_assoc(changeset, name, opts \\ [])

opts() ::
    :required,
    :required_message | "can't be blank"
    :invalid_message \ "invalid"
    :force_update_on_change \ true
    
    :with
    # method from associated module to use for cast.
    # {Author, :special_changeset, ["hello"]}
```

```elixir
User
|> Repo.get!(id)
|> Repo.preload(:addresses) # Only required when updating data
|> Ecto.Changeset.cast(params, [])
|> Ecto.Changeset.cast_assoc(
    :addresses, 
    with: &MyApp.Address.changeset/2
)
```

How params affect list of affected associations:

- `parameter does not contain ID`: 
    - the parameter data will be passed to MyApp.Address.changeset/2 with a new struct
    - insert operation
- `parameter contains an ID + no associated child with ID`: 
    - the parameter data will be passed to MyApp.Address.changeset/2 with a new struct
    - insert operation
- `parameter contains an ID + associated child with ID exists`
    - parameter data will be passed to MyApp.Address.changeset/2 with the existing struct
    - update operation
- `parameter does not contain ID` + `associated child with an ID exists`
    - `:on_replace` callback for that association will be invoked

### `on_replace`

Attempting to remove the association or embedded data via parent changeset, with `:on_replace`:

- `:raise` (default) - error raised
- `:mark_as_invalid` - error added to the parent changeset, and it will be marked as invalid
- `:nilify` (available for associations, not embeds) - sets owner reference column to nil. Allow the association to be cleared out so that it can be set later to a new value.
- `:update` (available for `has_one` and `belongs_to`). Update all the fields given to the changeset including the id for the association.
- `:delete` - removes the association or related data from the database. (NEVER USE IT directly, as it's too implicit). Do it explicitly:
```elixir
schema "comments" do
    field :body, :string
    field :delete_flag, :boolean, virtual: true
end
def changeset(comment, params) do
    cast(comment, params, [:body, :delete_flag])
    |> maybe_mark_for_deletion
end
defp maybe_mark_for_deletion(changeset) do
    if get_change(changeset, :delete_flag) do
        %{changeset | action: :delete}
    else
        changeset
    end
end
```

### `on_replace` with query-limited subset of affected associations
```elixir
query = from MyApp.Address, where: [country: ^edit_country]

User
|> Repo.get!(id)
|> Repo.preload(addresses: query)
|> Ecto.Changeset.cast(params, [])
|> Ecto.Changeset.cast_assoc(:addresses)
```

### Put assoc
```elixir
put_assoc(changeset, name, value)

value :: 
    structs # associating existing data
    | changesets # if validation is needed
    | maps # convenience for scripts
```
Used for:

- associating existing data (be careful - replacess all existing associations)
```elixir
# all associated tags are set at once (not one-by-one).
tags = Repo.all(
    from t in Tag, where: t.name in ^params["tags"]
)

post
|> Repo.preload(:tags)
|> Ecto.Changeset.cast(params, [:title]) 
# No need to allow :tags as we put them directly
|> Ecto.Changeset.put_assoc(:tags, tags)
# Explicitly set the tags
```

- simplify association in scripts, by passing `map` as value

```elixir
Ecto.Changeset.change(
  %Post{},
  title: "foo",
  comments: [
    %{body: "first"},
    %{body: "second"}
  ]
)
```

- set foreign key on a single association (BAD PRACTICE):
```elixir
# 1. Bad
%Comment{body: "bad example"}
|> Ecto.Changeset.change()
|> Ecto.Changeset.put_assoc(:post, post)
|> Repo.insert!()

# 2. Ok
%Comment{body: "ok example", post_id: post.id}
|> Repo.insert!()

# 3. Great
Ecto.build_assoc(post, :comments)
|> Ecto.Changeset.change(body: "great example")
|> Repo.insert!()
```

### Explicit control over Changeset action
```elixir
%{changeset | action: :ignore}
```