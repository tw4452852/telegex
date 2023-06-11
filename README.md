# Telegex

The new Telegram Bot API client features a unique and perfect implementation approach!

_😋 Three years ago I created this project, and three years later I redesigned it._

## Introduce

This library defines standardized APIs through code generation techniques using magical macro codes. These macros generate implementations from the data sourced from official documentation pages, which I parse into structured JSON data as macro inputs.

As a result, this library strictly adheres to the documentation standards. Due to its reliance on documentation data and code generation, adapting to new API versions is extremely easy. This ensures that it can effortlessly provide all the latest types and APIs while maintaining absolute correctness.

## Installation

Add Telegex to your mix.exs dependencies:

```elixir
def deps do
  [
    {:telegex, "~> 1.0.0-rc.8"},
  ]
end
```

>Note: `InputFile` with local file paths is not supported in the structured type of API parameters in version `1.0.0-rc.*`. Please use the file ID instead. This is temporary, and local file input will be supported upon the release of version 1.0.

## Configuration

Add bot token to the secret configuration file, like this:

```elixir
config :telegex, token: "<BOT_TOKEN>"
```

Specify the adapter for the HTTP client in the public configuration file:

```elixir
config :telegex, caller_adapter: Finch
```

Pass options to the adapter, such as timeout:

```elixir
config :telegex, caller_adapter: {Finch, [receive_timeout: 5 * 1000]}
```

You can also choose `HTTPoison` as the client. If using HTTPoison, set the corresponding adapter and timeout:

```elixir
config :telegex, caller_adapter: {HTTPoison, [recv_timeout: 5 * 1000]}
```

>Note: There are no standardized values for the `options` parameter here, as they directly relate to the HTTP client being used. The example above passes the raw options for the client library.

**Note: You need to manually add adapter-related libraries to the `deps`:**

- `Finch`: [`finch`](https://hex.pm/packages/finch), [`multipart`](https://hex.pm/packages/multipart)
- `HTTPoison`: [`httpoison`](https://hex.pm/packages/httpoison) (⚠️ sending local files is not supported, temporarily)

Don't have a client library you use? Tell me in issues!

## API call

All Bot APIs are located under the `Telegex` module, and these APIs fully comply with the required and optional parameters in the documentation, returning specific types (struct rather than map).

### getMe

```elixir
iex> Telegex.get_me
{:ok,
 %Telegex.Type.User{
   supports_inline_queries: false,
   can_read_all_group_messages: false,
   can_join_groups: true,
   added_to_attachment_menu: nil,
   is_premium: nil,
   language_code: nil,
   username: "telegex_dev_bot",
   last_name: nil,
   first_name: "Telegex Dev",
   is_bot: true,
   id: 6258629308
 }}
```

### getUpdates

```elixir
iex> Telegex.get_updates limit: 50
{:ok,
 [
   %Telegex.Type.Update{
     chat_join_request: nil,
     chat_member: nil,
     my_chat_member: nil,
     poll_answer: nil,
     poll: nil,
     pre_checkout_query: nil,
     shipping_query: nil,
     callback_query: nil,
     chosen_inline_result: nil,
     inline_query: nil,
     edited_channel_post: nil,
     channel_post: nil,
     edited_message: nil,
     message: %Telegex.Type.Message{
       reply_markup: nil,
       web_app_data: nil,
       # omitted part...
       new_chat_photo: nil,
       new_chat_title: nil,
       text: "Hello",
       # omitted part...
     },
     update_id: 929396006
   }
 ]}
```

### sendMessage

```elixir
iex> Telegex.send_message -1001486769003, "Hello!"
{:ok,
 %Telegex.Type.Message{
   venue: nil,
   chat: %Telegex.Type.Chat{
     # omitted part...
     title: "Dev test",
     type: "supergroup",
     id: -1001486769003
   },
   date: 1686111102,
   message_id: 12777,
   text: "Hello!",
   from: %Telegex.Type.User{
     supports_inline_queries: nil,
     can_read_all_group_messages: nil,
     can_join_groups: nil,
     added_to_attachment_menu: nil,
     is_premium: nil,
     language_code: nil,
     username: "telegex_dev_bot",
     last_name: nil,
     first_name: "Telegex Dev",
     is_bot: true,
     id: 6258629308
  }, 
  # omitted part...
 }}
```

## Polling mode

Polling is a simple and effective pattern that ensures messages within the same group arrive in order. Although it may not be the fastest, it is simple and reliable.

To work in polling mode:

1. Create a new module, like `YourProject.PollingHandler`
1. `Use Telegex.Polling.Handler`
1. Implement `on_boot/0` and `on_update/1` callback functions
1. Add your module to the supervision tree

Polling handler example:

```elixir
defmodule YourProject.PollingHandler do
  @moduledoc false

  use Telegex.Polling.Handler

  @impl true
  def on_boot do
    # delete any potential webhook
    {:ok, _} = Telegex.delete_webhook()
    # create configuration (can be empty, because there are default values)
    %Telegex.Polling.Config{}
    # you must return the `Telegex.Polling.Config` struct ↑
  end

  @impl true
  def on_update(update) do
    # consume the update

    :ok
  end
end

```

**Don't forget to add your module to the supervision tree.**

## Webhook mode

Add [`plug`](https://hex.pm/packages/plug) and [`remote_ip`](https://hex.pm/packages/remote_ip) to your application's deps because they are required for webhook mode.

You also need to configure adapters for hooks, which provide web services.

Based on [`Bandit`](https://hexdocs.pm/telegex/Telegex.Hook.Adapter.Bandit.html) - [`bandit`](https://hex.pm/packages/bandit)

```elixir
# add `bandit` to your dpes.
config :telegex, hook_adapter: Bandit
```

Based on [`Cowboy`](https://hexdocs.pm/telegex/Telegex.Hook.Adapter.Cowboy.html) - [`plug_cowboy`](https://hex.pm/packages/plug_cowboy)

```elixir
# add `plug_cowboy` to your dpes.
config :telegex, hook_adapter: Cowboy
```

To work in webhook mode:

1. Create a new module, like `YourProject.HookHandler`
1. `Use Telegex.Hook.Handler`
1. Implement `on_boot/0` and `on_update/1` callback functions
1. Add your module to the supervision tree

Hook handler example:

```elixir
defmodule YourProject.HookHandler do
  use Telegex.Hook.Handler

  @impl true
  def on_boot do
    # read some parameters from your env config
    env_config = Application.get_env(:your_porject, __MODULE__)
    # delete the webhook and set it again
    Telegex.delete_webhook()
    # set the webhook (url is required)
    Telegex.set_webhook(env_config[:webhook_url])
    # specify port for web server
    # port has a default value of 4000, but it may change with library upgrades
    %Telegex.Hook.Config{server_port: env_config[:server_port]}
    # you must return the `Telegex.Hook.Config` struct ↑
  end

  @impl true
  def on_update(update) do
  
    # consume the update
    :ok
  end
end
```

**Don't forget to add your module to the supervision tree.**

## Compatibility mode

You can create handlers for two modes and determine which one to start based on the configuration.

```elixir
updates_fetcher =
  if Application.get_env(:your_porject, :work_mode) == :webhook do
    YourProject.HookHandler
  else
    YourProject.PollingHandler
  end

children = [
  # omit other children
  updates_fetcher
]

opts = [strategy: :one_for_one, name: EchoBot.Supervisor]
Supervisor.start_link(children, opts)
```

## The end

Is there anything you don't understand about building a Telegram Bot? Have bot development needs? Welcome to contact me.
