## Polymorphic associations with many_to_many

Todo items can belong to:
- multiple projects (many-to-many),
- multiple todo lists (many-to-many)
```
TodoList *<-TodoList_Items->* TodoItems
Project *<-Project_Items->* TodoItems
```

**TIP**: it's not necessary to define schemas for `junction tables` (TodoList_Items, Project_Items).

### Schema definition
```elixir
defmodule MyApp.TodoList do
  use Ecto.Schema

  schema "todo_lists" do
    field :title
    many_to_many :todo_items, MyApp.TodoItem,
      join_through: "todo_list_items"
    timestamps()
  end

  def changeset(struct, params \\ %{}) do
    struct
    |> Ecto.Changeset.cast(params, [:title])
    |> Ecto.Changeset.cast_assoc(
      :todo_items,
      required: true
    )
  end
end

defmodule MyApp.Project do
  use Ecto.Schema

  schema "projects" do
    field :name
    many_to_many :todo_items, MyApp.TodoItem,
      join_through: "project_items"
    timestamps()
  end

  def changeset(struct, params \\ %{}) do
    struct
    |> Ecto.Changeset.cast(params, [:name])
    |> Ecto.Changeset.cast_assoc(
      :todo_items,
      required: true
    )
  end
end

defmodule MyApp.TodoItem do
  use Ecto.Schema

  schema "todo_items" do
    field :description
    timestamps()
  end

  def changeset(struct, params \\ %{}) do
    struct
    |> Ecto.Changeset.cast(params, [:description])
  end
end
```

### Migration
```elixir
create table("todo_lists")  do
  add :title
  timestamps()
end

create table("projects")  do
  add :name
  timestamps()
end

create table("todo_items")  do
  add :description
  timestamps()
end

# Primary key and timestamps are not required if
# using many_to_many without schemas
create table("todo_lists_items", primary_key: false) do
  add :todo_item_id, references(:todo_items)
  add :todo_list_id, references(:todo_lists)
  # timestamps()
end

# Primary key and timestamps are not required if
# using many_to_many without schemas
create table("projects_items", primary_key: false) do
  add :todo_item_id, references(:todo_items)
  add :project_id, references(:projects)
  # timestamps()
end
```