# Kno Elixir

Documentation and examples for using [trykno.com](https://trykno.com) in Elixir applications

- [Phoenix integration guide](#phoenix-integration-guide)

# Phoenix integration guide

*This guide requires Phoenix and Elixir.
The [Phoenix install guide](https://hexdocs.pm/phoenix/installation.html#content) can help you get both of these set up.*

## Setup project

Let's start a new project for a note taking app, which we shall call `my_notes`.
We will use Kno to authenticate users and start a session for them.

Kno is great for applications like this, it gives users a slick password free authentication that takes you only a few minutes to integrate.

If you want to try our device based authentication with your local app visit [trykno.app](https://trykno.app) on your mobile.

```sh
mix phx.new my_notes --no-webpack
```

Open up `mix.exs` and add HTTPoison and Jason as dependencies.

```elixir
defp deps do
  [
    # existing dependencies
    {:jason, "~> 1.1"},
    {:httpoison, "~> 1.6"},
  ]
end
```

Then run `mix deps.get` to pull the new dependencies.

## Configure API and site tokens

Next configure the tokens associated without your application.
Add the following code to `config/dev.exs`

```elixir
config :my_notes,
  kno_site_token: "kno_local_site_token",
  kno_api_token: "alpha.kno_local_site_token.kno_local_api_key"
```

*For production you will have keys unique to your application.
However you can use the tokens in this example as long as your example is running from localhost.*

**Please note that emails will be sent so you can test.
Ensure you use real emails so not to get blocked from using these credentials for local development**

## Display sign in/out buttons

Add the following code to `lib/my_notes_web/templates/layout/app.html.eex`, so that a user can sign in or out from any page.

```eex
<%= if authenticated?(@conn) do %>
  <%= link "Sign out", to: Routes.session_path(@conn, :sign_out) %>
<% else %>
  <%= form_for @conn, Routes.session_path(@conn, :sign_in), fn _form -> %>
    <script
      src="https://trykno.app/pass.js"
      data-site=<%= Application.get_env(:my_notes, :kno_site_token) %>>
    </script>
    <%= submit "Sign in" %>
  <% end %>
<% end %>
```

The `authenticated?` function is a helper that we define later.

For authenticated users a link to sign out is shown.
This link points to the `:sign_out` action found on the `MyNotesWeb.Session` controller.

Authenticated users see a button that will start the process of signing in.

Here we are using the [simple Kno integration](https://trykno.com/docs/#kno-now) as it is fastest way to get started.
When the form is submitted a sign in overlay is shown and once the user has been authenticated a pass token is added to the form.
This form, and the pass token within it is submitted to the`:sign_in` action on the `MyNotesWeb.Session` controller.

The helper function to tell if our user is authenticated is defined in `lib/my_notes_web/views/layout_view.ex`.

```elixir
def authenticated?(conn) do
  case Plug.Conn.get_session(conn, :persona_id) do
    persona_id when is_binary(persona_id) ->
      true

    nil ->
      false
  end
end
```

## Handle sign in/out actions

Update the router to point the actions we added to a new session controller.
In `lib/my_notes_web/router.ex` add the two routes to the top level `"/"` scope.

```elixir
scope "/", HelloWeb do
  pipe_through :browser

  get "/", PageController, :index
  post "/sign-in", SessionController, :sign_in
  get "/sign-out", SessionController, :sign_out
end
```

Create a session controller and add the two new actions.

```elixir
defmodule MyNotesWeb.SessionController do
  use MyNotesWeb, :controller

  def sign_in(conn, %{"knoToken" => token}) do
    persona_id = verify_token!(token)

    conn
    |> put_session(:persona_id, persona_id)
    |> redirect(to: "/")
  end

  def sign_out(conn, _params) do
    conn
    |> clear_session()
    |> redirect(to: "/")
  end

  defp verify_token!(token) do
    api_token = Application.get_env(:my_notes, :kno_api_token)

    url = "https://api.trykno.app/v0/pass"
    headers = [
      {"authorization", "Basic #{Base.encode64(api_token <> ":")}"},
      {"content-type", "application/json"}
    ]
    body = Jason.encode!(%{token: token})

    %{status_code: 200, body: response_body} = HTTPoison.post!(url, body, headers)
    %{"persona" => %{"id" => persona_id}} = Jason.decode!(response_body)
    persona_id
  end
end
```

## Protecting routes

Once a user has signed in they can Create Read Update & Delete (CRUD) notes that belong to them.
All of these paths can by defined at once.
Add the following code to `lib/my_notes_web/router.ex`.

```elixir
alias MyNotesWeb.Router.Helpers, as: Routes

scope "/notes", MyNotesWeb do
  pipe_through [:browser, :ensure_authenticated]

  resources "/", NoteController
end

def ensure_authenticated(conn, _) do
  case get_session(conn, :persona_id) do
    nil ->
      conn
      |> put_flash(:error, "You don't have permission to access that page")
      |> redirect(to: Routes.page_path(conn, :index))
      |> halt()

    persona_id when is_binary(persona_id) ->
      conn
      |> assign(:persona_id, persona_id)
  end
end
```

Every client request to interact with a note is first passed through the `ensure_authenticated` plug.
This plug checks that the session contains a persona_id.
For unauthenticated sessions the request is redirected with an error and halted.
If a persona_id was present it is added as an assign property on the plug, the request will then continue up the pipeline to be handled by the notes controller.

## Create a notes controller

Now it's time to create a controller for users to work with there notes.
This will live in `lib/my_notes_web/controllers/note_controller.ex`.

```elixir
defmodule MyNotesWeb.NoteController do
  use MyNotesWeb, :controller

  def index(conn, _params) do
    %{persona_id: persona_id} = conn.assigns

    notes = MyNotes.list_notes(persona_id)
    render(conn, "index.html", notes: notes)
  end

  def new(conn, _params) do
    %{persona_id: persona_id} = conn.assigns

    # Shrinkk this, the struct is a pain ideally just functions
    changeset = MyNotes.change_note(%MyNotes.Note{persona_id: persona_id})
    render(conn, "new.html", changeset: changeset)
  end

  def create(conn, %{"note" => note_params}) do
    %{persona_id: persona_id} = conn.assigns

    case MyNotes.create_note(note_params, persona_id) do
      {:ok, note} ->
        conn
        |> put_flash(:info, "Note created successfully.")
        |> redirect(to: Routes.note_path(conn, :show, note))

      {:error, %Ecto.Changeset{} = changeset} ->
        render(conn, "new.html", changeset: changeset)
    end
  end

  def show(conn, %{"id" => id}) do
    %{persona_id: persona_id} = conn.assigns

    note = MyNotes.get_note!(id, persona_id)
    render(conn, "show.html", note: note)
  end

  def edit(conn, %{"id" => id}) do
    %{persona_id: persona_id} = conn.assigns

    note = MyNotes.get_note!(id, persona_id)
    changeset = MyNotes.change_note(note)
    render(conn, "edit.html", note: note, changeset: changeset)
  end

  def update(conn, %{"id" => id, "note" => note_params}) do
    %{persona_id: persona_id} = conn.assigns

    note = MyNotes.get_note!(id, persona_id)

    case MyNotes.update_note(note, note_params) do
      {:ok, note} ->
        conn
        |> put_flash(:info, "Note updated successfully.")
        |> redirect(to: Routes.note_path(conn, :show, note))

      {:error, %Ecto.Changeset{} = changeset} ->
        render(conn, "edit.html", note: note, changeset: changeset)
    end
  end

  def delete(conn, %{"id" => id}) do
    %{persona_id: persona_id} = conn.assigns

    note = MyNotes.get_note!(id, persona_id)
    {:ok, _note} = MyNotes.delete_note(note)

    conn
    |> put_flash(:info, "Note deleted successfully.")
    |> redirect(to: Routes.note_path(conn, :index))
  end
end
```
*Should this be a Note or Notes controller?*

Every action can extract the persona_id from the assign property of the conn,
relying on authentication to be handled by the ensure_authenticated plug.

The `MyNotes` module is the interface to the business logic of managing notes.
For this guide we do not need to worry about the implementation.
However if you want to see how to do it using Ecto and the database take a look at [example]('examples/phoenix-integration') in this repo.

<!--
## Saving notes in the database

Add a migration to create a notes table so that the application can save notes in the database.

```shell
mix ecto.gen.migration create_notes
```

In the generated file at `/priv/repo/migrations/[timestamp]_create_notes.exs` create a table for notes with a title content and persona_id.

```elixir
defmodule MyNotes.Repo.Migrations.CreateNotes do
  use Ecto.Migration

  def change do
    create table(:notes) do
      add :persona_id, :binary_id, null: false

      add :title, :text, null: false
      add :content, :text, null: false
    end

    create index(:notes, :persona_id)
  end
end
```

Create the file `lib/my_notes/note.ex` in which we will add the Ecto model for accessing notes in the database.

```elixir
defmodule MyNotes.Note do
  use Ecto.Schema

  schema "notes" do
    field :persona_id, :binary_id
    field :title, :string
    field :content, :string
  end

  @doc false
  def changeset(note, attrs) do
    import Ecto.Changeset

    note
    |> cast(attrs, [:title, :content])
    |> validate_required([:persona_id, :title, :content])
  end
end
```




Keep list of notes because its one persona to n where profile will be one to oneet

    note
    |> cast(attrs, [:title, :content])
    |> validate_required([:persona_id, :title, :content])
  end
end
```

```
defmodule MyNotes do

  import Ecto.Query, warn: false

  alias MyNotes.Note
  alias MyNotes.Repo

  @doc """
  Returns the list of notes for a given persona id.
  """
  def list_notes(persona_id) when is_binary(persona_id) do
    from(n in Note, where: n.persona_id == ^persona_id, order_by: :inserted_at)
    |> Repo.all()
  end

  @doc """
  Gets a single note owned by a persona.
  """
  def get_note!(id, persona_id), do: Repo.get_by!(Note, id: id, persona_id: persona_id)

  @doc """
  Creates a note for a persona.
  """
  def create_note(attrs, persona_id) do
    %Note{persona_id: persona_id}
    |> Note.changeset(attrs)
    |> Repo.insert()
  end

  @doc """
  Updates an existing note.
  """
  def update_note(%Note{} = note, attrs) do
    note
    |> Note.changeset(attrs)
    |> Repo.update()
  end

  @doc """
  Deletes a Note.
  """
  def delete_note(%Note{} = note) do
    Repo.delete(note)
  end

  @doc """
  Returns an `%Ecto.Changeset{}` for tracking note changes.
  """
  def change_note(%Note{} = note) do
    Note.changeset(note, %{})
  end
end
```

```
defmodule MyNotesWeb.NoteView do
  use MyNotesWeb, :view
end
```

```
<h1>Your Notes</h1>

<table>
  <thead>
    <tr>
      <th>Title</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
  <%= for note <- @notes do %>
    <tr>
      <td><%= note.title %></td>
      <td>
        <%= link "Show", to: Routes.note_path(@conn, :show, note) %> &middot;
        <%= link "Edit", to: Routes.note_path(@conn, :edit, note) %> &middot;
        <%= link "Delete", to: Routes.note_path(@conn, :delete, note), method: :delete, data: [confirm: "Are you sure?"] %>
      </td>
    </tr>
  <% end %>
  </tbody>
</table>

<span><%= link "Create Note", to: Routes.note_path(@conn, :new) %></span>
```

Keep list of notes because its one persona to n where profile will be one to one -->