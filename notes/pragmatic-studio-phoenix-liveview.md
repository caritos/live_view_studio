# What is LiveView?

- LiveView is a library included in every Phoenix app that unifies everything and gives a simple yet powerful programming model.
- server rendered html over websockets
- runs on phoenix server that can scale that can handle millions of websocket connections
- built-in presence tracking
- built-in pubsub system for broadcasting real-time updates to LiveViews

# Setup

# Button Clicks

- add route to LiveView process in router.ex

```
scope "/", LiveviewStudioWeb do
   live "/light", LightLive
```

- LiveView files exists in lib/live_view_studio_web/live/
- light_live.ex as a sample LiveView module

```
defmodule LiveViewStudioWeb.LightLive do
  use LiveViewStudioWeb, :live_view

  def mount(_params, _session, socket) do
    socket = ssign(socket, brightness: 10)
    {:ok, socket}
  end

  def render(assigns) do
    ~H"""
    <h1>Front Port Light</h1>
    <div id="light">
      <div class="meter">
        <span style={"width: #{@brightness}%"}>
          <%= @brightness %>%ß
        </span>
      <div>
    </div>
    """
  end

  # handle_event
end
```

- mount function is invoked first and assigns the LiveViews initial state
- render is called to render the html
  - validates the structure of templates.
- ex tags can only be used in the body of an html tag, inside the tag itself curlys' must be used.
- handles when the button on is pressed

```
def render(assigns) do
  <button phx-click="off">
    <img src="/images/light-off.svg"
  </button>
end

def handle_event("on", _, socket) do
  socket = assigns(socket, brightness: 100)
  {:noreply, socket}
end
```

- shortcut is to use the update function instead of the assigns function

```
def handle_event("up", _, socket) do
  socket = update(socket, :brightness, &(&1 + 10))
end

```

- handle_event functions to handle different events
  - assigns to assign a variable
  - update to updatte a variable

# LiveView Life Cycle

- when the initial page is loaded, app.js is loaded as well. this creates a websocket connection.
- after the creation of a websocket connection, a staeful liveview process is instanstiated and mount is invoked again
- render is invoked again
- raw html is not delivered, html page is separated by static vs dynamic parts,
- only changed variables are returned to the liveview page
- layout files are located in lib/live_view_studio_web/layouts/
  - this is where all the outer html comes from

# Dynamic Form

- in mount, setup the initial variables

```
def mount(_params, _session, socket) do
  socket =
    assign(socket,
      length: "0",
      width: "0",
      depth: "0",
      weight: 0.0,
      price: nil
    )
  {:ok, socket}
end
```

- since the values are inside the html tag, we need to use braces

```
<input type="number" name="length" value={@length} />
```

- since it is inside a tag body, we can use a heex tag

```
<div class="weight">
  You need <%= @weight %> pounds of sound
</div>
```

- convert string to number using the number package

```
import Number.Currency
<%= number_to_currency(@price) %>
```

- display the price conditionally

```
<%= if @price do %>
  <div class="quote">
    <%= number_to_currency(@price) %>
  </div>
<% end %>

```

- or use the shortcut

```
<div :if={@price} class="quote">
  ...
<div>
```

- it's always emit an event and then handle the event

```
<form phx-chhange="calculate">

def handle_event("calculate", params, socket) do
  %{"length" => l, "width" => w, "depth" => d} = params
  weight = Sandbox.calculate_weight(l,w,d)
  socket = assign(socket,
    weight: weight,
    length: l,
    width: w,
    depth: d)
  {:noreply, socket}
end
```

- need to handle the submission of a button

```
<form phx-change="calculate" phx-submit="get-quote">

def handle_even("get-quote", _, socket) do
  price = Sandbox.calculate_price(socket.assigns.weight)
  socket = assigns(socket, price: price)
  {:noreply, socket}
end

```

- set the price to nil during the caculate function to remove the quote

# Dashboard

- internal messages generate the UI updates
- create a refresh event when the button is clicked

```
<button phx-click="refresh">
  <img src="/images/refresh.svg"/>Refresh
</button>

def handle_event("refresh", _, socket) do
    {:noreply, assign_stats(socket)}
end

defp assign_stats(socket) do
  assign(socket,
    new_orders: Sales.new_orders(),
    sale_amount: Sales.sales_mount(),
    satisfaction: sales.satisfaction()
  )
end
```

- need the liveview process to send itself an event intermittenttly
- messages sent by elixir process are handled by handle_info

```
def mount(_params, _session, socket) do
  if connected?(socket) do
    :timer.send_interval(1000, self(), :tick)
  end
end

def handle_info(:tick, socket) do
  {:noreply, assign_stats(socket)}
end
```

# Search

- in heex templates, you can do a shortcut

```
<%= for flight <- @flights do %>
```

```
<li :for={flight <- @flights}>
```

- make the website interact

```
<form phx-submit="search">

def handle_event("search", %{"airport" => airport}, socket) do
  socket = assign(socket,
    airports: airport,
    flights: Flights_search_by_airport(airport)
  )
  {:noreply, socket}
end
```

- add a loading

```

send(self(), {:run_search, airport})

<dif :if={@loading} class="loader">Loading...</div>
```

# Autocomplete

```
<form phx-submit="search" phx-change="suggest">

def handle_event("suggest", %{"airport" => prefix}, socket) do
  matches = Airports.suggest(prefix)
  {:noreply, assign(socket, matches: matches}
end

<datalist id="matches">
  <option :for={{code, name} <- @matches} value={code}>
    <%= name %>
  </option>
</datalist>
```

- limit how many times a query is made using phx-debounce

```
<form phx-submit="search" phx-change="suggest">
  <input
    type="text"
    name="airport"
    value={@airport}
    placeholder="Airport Code"
    autofocus
    autocomplete="off"
    readonly={@loading}
    list="matches"
    phx-debounce="1000"
```

# Filtering

- look infto migrations files for the database migration file
- priv/repo/migrations
- connects to ecto
- lib/live_view_studio/boats/boats.ex for database schema file

```
defmodule LiveViewStudio.Boats.Boat do
  use Ecto.Schema
  import Ecto.Changeset

  schema "boats" do
    field :model, :string
    field :type, :string
    field :price, :string
    field :image, :string
  end

  def changeset(boat,attrs) do
    boat
    |> cast(attrs, [:model, :type, :price, :image])
    |> validate_required([:model, :type, :price, :image])
  end
end
```

- standard for managing a datta stored in the database using ecto

```
from(Boat)
|> filter_by_type(filter)
|> filter_by_prices(filter)
|> Repo.all()
```

- form has two ways to filter

```
<select name="type">
  <%= Phoenix.HTML.Form.options_for_select(
    type_options()),
    @filter.type
    ) %>
</select>

<%= for price <- ["$"< "$$", "$$$"] do %>
<input
  type="checkbox"
  name="pricesp[]"
  value={price}
  id={price}
  checked={price in @filter.prices}
/>

```

- to handle any changes in the form

```
<form phx-change="filtter">

def handle_event("filter", %{"type" => type, "prices" > prices}, socket) do
  filter = %{type: type, prices: prices}
  boats = Boats.list_boats(filter)
  {:noreply, assign(socket, boats: boats, filter: filter)}
```

- don't want to keep the boats in memory

```
def mount(_params, _session, socket) do
  ...
  {:ok, socket, , temporarty_asssigns: [boats: []]}
end
```

# Function Components

- function components are like banners that can be used throughout the website, not just a particular webpage
- takes in an "assigns" as a parameter and returns a heex template
- to access a local function component

```
<.promo />
```

- the default slot for the function component
- use inner block to get the text

```
def promo(assigns) do
  ~H"""
  <div class="promo">
    <%= render_slot(@inner_block) %>
  </div>
  """
end

<.promo>
  Save 25% on rentals!
</.promo>
```

- named slots

```
<:legal>
  Excluding weekends
</:legal>

<%= render_slot(@legal) %>
```

- you can pass parameters in function components

```
<.prompo expirations={1}>

<div class="expiration">
  Deal expires in <%= @expirations %>
</div>
```

# Live Navigation

- phoenix context generator
  - need more research about this topic
  - it seems to have been passed voer
- add a '~p' lets phoenix verify that a route exists

```
href={~p"/servers?id=#{server.id}"}
```

- use .link

```
<.link
  :for={server <- @servers}
  href={~p"/servers?#{[id: server]}"}
  class={if serer == @selected_server, do: "selectec"}
>
```

- after updating the url, how can you make the liveview page respond? via handle_params

```
def handle_params(%{"id" => id}, _url, socket) do
  server = Servers.get_server!(id)
  {:noreply, assign(socket, selected_server:server)}

end

def handle_params(_, _uri, socket) do

  {:noreply, assign(socket, selected_server: hd(socket.assigns.server))}
end

```

- losing the coffee count since the variable is getting lost between process
- use the patch attribute instead of link

```
<.link
  :for={server <- @servers}
  patch={~p"/servers?{[id: server]}"}
>
```

# Sorting

- when the render function is not defined in the LiveView file, it looks for a heex template instead.
- live/donations_live.html.heex
- with the ttemplate file, you don't use a heex sigle (``````)
- need to update the headers to include a link (patch) to update the liveview url

```
<.link patch={~p"/donations?#{%{sort_by: :item, sort_order: asc}}"}
```

```
def handle_params(params, _uri, socket) do
  sort_by = (params["sort_by"] || id) |> String.to_atom()
  sort_order = (params["sort_order"] || "asc") |> String.to_atom()

  options = %{
    sort_by: sort_by,
    sort_order: sort_order
  }

  donations = Donations.list_donations(options)

  socket = assign(socket, donations: donations)

  {:noreply, socket}
end
```

- need to toggle the sort order

```
defp next_sort_order(sort_order) do
  case sort_order do
    :asc -> :desc
    :desc -> :asc
  end
end
```

# Pagination

- use push_patch to update the live_view server side

```
def handle_event("select-per-page", %{"per-page" => per_page}, socket) do
  params = %{socket.assigns.options | per_page: page_page}
  socket = push_patch(socket, to: ~p"/donations?#{params}")
  {:noreply, socket}
end

```

# Live Ecto Forms and Lists

- a changeset is Ecto's way of creating an item in the db
- use .form to use with a changeset

```
<.form for={@form} phx-submit="save">
  <.input field={@form[:name]} placeholder="Name" autocomplete="offf" />
  <.input field={@form[:phone]} type="tel" placeholder="Phone" autocomplete="off"
  <.button phx-disable-with="Saving...">
    Check in
  </.button>
</.form>

def handle_event("save", %{"volunteer" => volunteer_param}, socket) do

  case Volunteers.create_volunteer(volunteer_params) do
    {:ok, voluneter} ->
      socket = update(socket, :volunterrs, fn volunteers -> [volunteer | volunteers| end ])
      changset = Vounteers.change_vuluneter({%Voluntter()})
      {:noreply, assign(ocket, :form, to_form(changeset))}
    {:error, changeset} ->
      {:noreply, assign(ocket, :form, to_form(changeset))}
  end


end
```

# Live Validations

- need to include a phx-change to validate form entry
- need to explicitly change the action of the changeset
- add debouncing

```
<.form for={@form} phx-submit="save" phx-change="validate">
  <.input phx-debounce="2000"
  <.input phx-debounce="blur"

def handle_event("validate", %{"volunteer" => volunteer_params}, socket) do
  changeset =
    %Volunteer{}
 |> Volunteers.change_volunterr(volunteer_params)
 |> Map.put(:action, :validate)
  {:noreply, assign(socket, :form, to_form(changeset))}
end
```

# Streams

- dom id is required in the client side for identification
- use stream
- in mount method, use the streaam function

```
socket =
    socket
    |> stream(:volunteers, volunteers)
    |> assign(:form, to_form(changeset))
```

- render then using the @streams

```
<div id="volunteers" phx-update="stream">
  <div
    :for={{volunteer_id, volunteer} <- streams.volunteers}
    class=={"volunteer #{if volunteer.checked_out, do: "out}"}
```

# Toggling State

```
<button phx-click="toggle-status" phx-value-id={volunteer.id}>

def handle_event(toggle-status, %{id -> id}, socket) do
  volunteer = Volunteers.get_volunteer!(id)

  {:od, volunteer} =
    Volunteers.update_vuolunteer(
     voluntteer,
     %{checked_out: !volunteer.checked_out}
  )

  {:noreply, stream_insert(socket, :volunteers, volunter)}
)
end
```

# Live Components

todo
need to redo this later

# Real-Time Updates

- phoenix pubsub

```
def subscribe do
  Phoenix.PubSub.subscribe(LiveViewStudio.PubSub, "volunteers")
end


def broadcase(message) do
  Phoenix.Pubsub.broadcase(LiveViewStudio.PubSub, "volunteers", message)
end
```

# Authenticating LiveViews

- mix phx.gen.auth Accounts User users
- use LiveView

- need to put it in a new scope
- add on_mount hook

- any authentication checks must be performmed in two places

```
scope "/", LiveViewStudioWeb do
 pipe_through [:browser, :require_authenticated_user]

 live "/topsecret", TopSecretLive
end
```

- use on_mount to access the user

```
on_mount({LiveViewStudioWeb.UserAuth, :ensure_authenticated})
```

# Live Sessions

```
scope "/", LiveViewStudioWeb do
  pipe_through [:browser, :require_authenticated_user]

  live_session :authenticated do
    on_mount: {LiveViewStudioWeb.UserAuth, :ensure_autehtciated} do
    live "topsecret", TopSecretLive
    live "/present", PresenceLive
  end
end
```

# Presence Tracking

- mix phx.gen.presence
- add to application.ex
  - LiveViewStudioWeb.Presence
- use PubSub to track incoming and outgoing

```
def mount(_params, _session, socket) do
  %{current_user: current_user} = socket.assigns

  if connected?(socket) do
    {:ok, _} = Presence.track(self(), @topic, current_user.id, %{
      username: current_user.email |> Sttring.split("@) |> hd(),
      is_Playing: false")

    })
  end
end
```

# JS Commands

# JS Hooks

# Key Events

- live view has first class support for key events

# File Uploads: UI

- use streams

# File Uploads: Server

# File Uploads: Cloud
