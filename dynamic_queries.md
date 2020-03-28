## Dynamic macro

Used to decouple the processing of parameters from the query generation

**Tip:** use `:as + keyword match` instead of positional arguments: 
`q |> where([_, _, a], a > 10)`, do:
`q |> where([author: a], a > 10)`

- build dynamic expressions that are later interpolated into the query
- dynamic expressions can also be interpolated into dynamic expressions

```elixir
query =
  Post
  |> where(^where)
  |> order_by(^order_by)

query =
  if published_at = params["published_at"] do
    where(query, [p], p.published_at < ^published_at)
  else
    query
  end

#######

where = [author: "José", category: "Elixir"]
order_by = [desc: :published_at]

filter_published_at =
  if published_at = params["published_at"] do
    dynamic([p], p.published_at < ^published_at)
  else
    true
  end

Post
|> where(^where)
|> where(^filter_published_at)
|> order_by(^order_by)
```

### `dynamic` example
```elixir
def filter(params) do
  Post
  |> order_by(^filter_order_by(params["order_by"]))
  |> where(^filter_where(params))
end

def filter_order_by("published_at_desc"),
  do: dynamic([p], desc: p.published_at)

def filter_order_by("published_at"),
  do: dynamic([p], p.published_at)

def filter_order_by(_),
  do: []

def filter_where(params) do
  Enum.reduce(params, dynamic(true), fn
    {"author", value}, dynamic ->
      dynamic([p], ^dynamic and p.author == ^value)

    {"category", value}, dynamic ->
      dynamic([p], ^dynamic and p.category == ^value)

    {"published_at", value}, dynamic ->
      dynamic([p], ^dynamic and p.published_at > ^value)

    {_, _}, dynamic ->
      # Not a where parameter
      dynamic
  end)
end
```

### `dynamic` and tests

```elixir
test "filter published at based on the given date" do
  assert dynamic_match?(
           filter_where(%{}),
           "true"
         )

  assert dynamic_match?(
           filter_where(%{"published_at" => "2010-04-17"}),
           "true and q.published_at > ^\"2010-04-17\""
         )
end

defp dynamic_match?(dynamic, string) do
  inspect(dynamic) == "dynamic([q], #{string})"
end
```

### `dynamic` and joins
```elixir 
def filter(params) do
  Post
  # 1. Add named join binding
  |> join([p], assoc(p, :authors), as: :authors)
  |> order_by(^filter_order_by(params["order_by"]))
  |> where(^filter_where(params))
end

# 2. Returned dynamic with join binding
def filter_order_by("published_at_desc"),
  do: dynamic([p], desc: p.published_at)

def filter_order_by("published_at"),
  do: dynamic([p], p.published_at)

def filter_order_by("author_name_desc"),
  # Note the use of [authors: a]
  do: dynamic([authors: a], desc: a.name)

def filter_order_by("author_name"),
  do: dynamic([authors: a], a.name)

def filter_order_by(_),
  do: []

# 3. Change the authors clause inside reduce
# (dynamic inside dynamic example)
def filter_where(params) do
  Enum.reduce(params, dynamic(true), fn
    {"author", value}, dynamic ->
      dynamic([authors: a], ^dynamic and a.name == ^value)

    {"category", value}, dynamic ->
      dynamic([p], ^dynamic and p.category == ^value)

    {"published_at", value}, dynamic ->
      dynamic([p], ^dynamic and p.published_at > ^value)

    {_, _}, dynamic ->
      # Not a where parameter
      dynamic
  end)
end
```