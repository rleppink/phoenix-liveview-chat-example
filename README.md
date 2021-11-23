# LiveviewChat

![Libraries.io dependency status for GitHub repo](https://img.shields.io/librariesio/github/dwyl/phoenix-liveview-chat-example)

## Create Phoenix LiveView App

### Initialisation

Let's start by creating the new `liveview_chat` Phoenix application.

```sh
mix phx.new liveview_chat --no-mailer --no-dashboard
```

We don't need mail or dashboard features. You can learn more about creating
a new Phoenix application with `mix help phx.new`

Run `mix deps.get` to retreive the dependencies, then make sure you have
the `liveview_chat_dev` Postgres database available.
You should now be able to start the application with `mix phx.server`:

![phx.server](https://user-images.githubusercontent.com/6057298/142623156-ab767540-2561-43e3-bc87-1c4f89778d21.png)

### LiveView Route, Controller and Template

Open `lib/liveview_chat_web/router.ex` file to remove the current default `PageController`
controller and add instead a `MessageLive` controller:


```elixir
scope "/", LiveviewChatWeb do
  pipe_through :browser

  live "/", MessageLive
end
```

and create the `live` folder and the  controller at `lib/liveview_chat_web/live/message_live.ex`:

```elixir
defmodule LiveviewChatWeb.MessageLive do
  use LiveviewChatWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok, socket}
  end

  def render(assigns) do
    LiveviewChatWeb.MessageView.render("messages.html", assigns)
  end
end
```

A liveView controller requires the function `mount` and `render` to be defined.
To keep the controller simple we are just retruning the {:ok, socket} tuple
without any changes and the render view call the `message.html.heex` template.
So let's create now the `MessageView` module and the `message` template:

Similar to normal Phoenix view create the `lib/liveview_chat_web/views/message_view.ex`
file:

```elixir
defmodule LiveviewChatWeb.MessageView do
  use LiveviewChatWeb, :view
end
```

Then the template at `lib/liveview_chat_web/templates/message/message.html.heex`:

```heex
<h1>LiveView Message Page</h1>
```

Finally to make the root layout simpler. Update the `body` content
of the `lib/liveview_chat_web/templates/layout/root.html.heex` to: 

```
<body>
    <header>
      <section class="container">
        <h1>LiveView Chat Example</h1>
      </section>
    </header>
    <%= @inner_content %>
</body>
```


Now if you refresh the page you should see the following:

![live view page](https://user-images.githubusercontent.com/6057298/142659332-bc15ed66-195a-482f-8925-ec6c57c478c0.png)

At this point we want to update the tests!
Create the `test/liveview_chat_web/live` folder and the `message_live_test.exs`:

```elixir
defmodule LiveviewChatWeb.MessageLiveTest do
  use LiveviewChatWeb.ConnCase
  import Phoenix.LiveViewTest

  test "disconnected and connected mount", %{conn: conn} do
    conn = get(conn, "/")
    assert html_response(conn, 200) =~ "<h1>LiveView Message Page</h1>"

    {:ok, _view, _html} = live(conn)
  end
end

```

We are testing the `/` endpoint is accessible when the socket is not yet connected,
then when it is with the `live`function.

See also the [LiveViewTest module](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveViewTest.html)
for more information about testing and liveView.

Finally you can delete all the default generated code linked to the `PageController`.
- rm test/liveview_chat_web/controllers/page_controller_test.exs
- rm lib/liveview_chat_web/controllers/page_controller.ex
- rm test/liveview_chat_web/views/page_view_test.exs
- rm lib/liveview_chat_web/views/page_view.ex
- rm -r lib/liveview_chat_web/templates/page

You can now run the test with `mix test` command:

![image](https://user-images.githubusercontent.com/6057298/142856124-5c2d9cc6-9208-4567-b781-0b46081cfed1.png)


### Migration and Schema

Now that we have the liveView structure defined,
we can start to focus on creating messages.
The database will save the message and the name of the user.
So we can create a new schema and migration:

```sh
mix phx.gen.schema Message messages name:string message:string
```

and don't forget to run `mix ecto.migrate` to create the new `messages` table.

We can now udpate the `Message` schema to add functions for creating
new messages and listing the existing messages. We'll update also the changeset
to add requirements and validations on the message text.
Open the `lib/liveview_chat/message.ex` file and upadate the code with the following:

```elixir
defmodule LiveviewChat.Message do
  use Ecto.Schema
  import Ecto.Changeset
  import Ecto.Query
  alias LiveviewChat.Repo
  alias __MODULE__

  schema "messages" do
    field :message, :string
    field :name, :string

    timestamps()
  end

  @doc false
  def changeset(message, attrs) do
    message
    |> cast(attrs, [:name, :message])
    |> validate_required([:name, :message])
    |> validate_length(:message, min: 2)
  end

  def create_message(attrs) do
    %Message{}
    |> changeset(attrs)
    |> Repo.insert()
  end

  def list_messages do
    Message
    |> limit(20)
    |> order_by(desc: :inserted_at)
    |> Repo.all()
  end
end
```

We have added the `validate_length` function on the message input to make
sure messages have at least 2 characters. This is just an example to show how
the changeset works with the form on the liveView page.

We then created the `create_message` and `list_messages` functions.
Similar to [phoenix-chat-example](https://github.com/dwyl/phoenix-chat-example/)
we limit the number of messages returned to the latest 20.


We can now update the `lib/liveview_chat_web/live/message_live.ex` file to use
the `list_messages` function:

```elixir
def mount(_params, _session, socket) do
  messages = Message.list_messages() |> Enum.reverse()
  changeset = Message.changeset(%Message{}, %{})
  {:ok, assign(socket, messages: messages, changeset: changeset)}
end
```

We get the list of messages and we create a changeset that we'll use for the
message form.
We then [assign](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html#assign/2)
the changeset and the messages to the socket which will display them on the liveView page.

If we now update the `message.htlm.heex` file to:

```html
<ul id='msg-list'>
   <%= for message <- @messages do %>
     <li id={message.id}>
       <b><%= message.name %>:</b>
       <%= message.message %>
     </li>
   <% end %>
</ul>

<.form let={f} for={@changeset}, id="form", phx-submit="new_message">
   <%= text_input f, :name, id: "name", placeholder: "Your name", autofocus: "true"  %>
   <%= error_tag f, :name %>

   <%= text_input f, :message, id: "msg", placeholder: "Your message"  %>
   <%= error_tag f, :message %>

   <%= submit "Send"%>
</.form>
```




We should see the following page:

![image](https://user-images.githubusercontent.com/6057298/142882923-db490aea-5af6-49d4-9e45-38c75d05e234.png)


the `<.form></.form>` syntax is how to use the form [function component](https://hexdocs.pm/phoenix_live_view/Phoenix.Component.html#content).
> A function component is any function that receives an assigns map as argument and returns a rendered struct built with the ~H sigil.





Finally let's make sure the test are still passing by updating the `assert` to:

```elixir
assert html_response(conn, 200) =~ "<h1>LiveView Chat Example</h1>"
```

As we have deleted the `LiveView Message Page` h1 title, we can test instead
the title in the root layout and make sure the page is still displayed correctly.

## Handle events

At the moment if we submit the form nothing happen.
If we look at the server log, we see the following:

```sh
** (UndefinedFunctionError) function LiveviewChatWeb.MessageLive.handle_event/3 is undefined or private
    (liveview_chat 0.1.0) LiveviewChatWeb.MessageLive.handle_event("new_message", %{"_csrf_token" => "fyVPIls_XRBuGwlkMhxsFAciRRkpAVUOLW5k4UoR7JF1uZ5z2Dundigv", "message" => %{"message" => "", "name" => ""}}, #Phoenix.LiveView.Socket
```

On submit the form is creating a new event defined with `phx-submit`:

```elixir
<.form let={f} for={@changeset}, id="form", phx-submit="new_message">
```

However this event is not managed on the server yet, we can fix this by adding the
`handle_event` function in `lib/liveview_chat_web/live/message_live.ex`:

```elixir
def handle_event("new_message", %{"message" => params}, socket) do
  case Message.create_message(params) do
    {:error, changeset} ->
      {:noreply, assign(socket, changeset: changeset)}

    {:ok, _message} ->
      {:noreply, socket}
    end
end
```

The `create_message` function is called with the values from the form.
If an error occurs while trying to save the information in the database,
for example the changeset can returns an error if the name or the message is
empty or if the message is too short, the changeset is assign again to the socket.
This will allow the form to display the error information:

![image](https://user-images.githubusercontent.com/6057298/142921586-2ed0e7b4-c2a1-4cd2-ab87-154ff4e9f4d8.png)

If the message is saved without any errors the tuple `{:noreply, socket}` is returned.
If you reload the page you should be able to see the messages created:

![image](https://user-images.githubusercontent.com/6057298/142921871-2feb20c2-906e-4640-8781-f8ea776dc05b.png)

Now the form is displayed we can add the following tests:


```elixir
  test "name can't be blank", %{conn: conn} do
    {:ok, view, _html} = live(conn, "/")

    assert view
           |> form("#form", message: %{name: "", message: "hello"})
           |> render_submit() =~ html_escape("can't be blank")
  end

  test "message", %{conn: conn} do
    {:ok, view, _html} = live(conn, "/")

    assert view
           |> form("#form", message: %{name: "Simon", message: ""})
           |> render_submit() =~ html_escape("can't be blank")
  end

  test "minimum message length", %{conn: conn} do
    {:ok, view, _html} = live(conn, "/")

    assert view
           |> form("#form", message: %{name: "Simon", message: "h"})
           |> render_submit() =~ "should be at least 2 character(s)"
  end
```

We are using the [form](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveViewTest.html#form/3) function to select the form and trigger
the submit event with different values for the name and the message.
We are testing that errors are properly displayed.


## PubSub

Instead of having to reload the page to see the new created messages,
we can use [PubSub](https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.html) 
to notice all connected clients that a new message has been created.


