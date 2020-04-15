## TL;DR
Creating an elixir project for docker that communicates through the HackerNews API module and RabbitMQ
### Article Link
https://akoutmos.com/post/broadway-rabbitmq-and-the-rise-of-elixir/
https://akoutmos.com/post/broadway-rabbitmq-and-the-rise-of-elixir-two/
https://link.medium.com/RqYhnjEgq3
https://shopgun.github.io/graphql-erlang-tutorial/#_setting_up_an_initial_mnesia_schema
### Author
Alex Koutmos
## Key Takeaways
* Use tools like Grafana and Prometheous to monitor your client
* Create batch queques to fetch topics from RabbitMQ

## Useful Code Snippets
Docker-compose.yaml
```yaml
version: '3.7'

services:
  cadvisor:
    image: google/cadvisor:v0.33.0
    ports:
      - '8080:8080'
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

  rabbitmq:
    image: rabbitmq:3.8
    ports:
      - '5672:5672'
      - '15672:15672'
      - '15692:15692'
    volumes:
      - ./docker/rabbitmq/plugins:/etc/rabbitmq/enabled_plugins
      - ./docker/rabbitmq/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - rabbitmq-data:/var/lib/rabbitmq

  postgres:
    image: postgres:12.0
    ports:
      - '5432:5432'
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres

  postgres_exporter:
    image: wrouesnel/postgres_exporter:v0.7.0
    ports:
      - '9187:9187'
    depends_on:
      - postgres
    environment:
      DATA_SOURCE_USER: postgres
      DATA_SOURCE_PASS: postgres
      DATA_SOURCE_URI: postgres:5432/?sslmode=disable

  grafana:
    image: grafana/grafana:6.4.4
    depends_on:
      - prometheus
    ports:
      - '3000:3000'
    volumes:
      - grafana-data:/var/lib/grafana
      - ./docker/grafana/:/etc/grafana/provisioning/
    env_file:
      - ./docker/grafana/.env

  prometheus:
    image: prom/prometheus:v2.13.0
    ports:
      - '9090:9090'
    volumes:
      - ./docker/prometheus/:/etc/prometheus/
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/config.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'

volumes:
  postgres-data: {}
  rabbitmq-data: {}
  prometheus-data: {}
  grafana-data: {}

```

```elixir
use Mix.Config

config :elixir_popularity, ecto_repos: [ElixirPopularity.Repo]

config :elixir_popularity, ElixirPopularity.Repo,
  database: "elixir_popularity_repo",
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  log: false

config :logger, :console, format: "[$level] $message\n"
```
HackerNews
```elixir
defmodule ElixirPopularity.HackernewsItem do
  alias __MODULE__

  defstruct text: nil, type: nil, title: nil, time: nil

  def create(text, type, title, unix_time) do
    %HackernewsItem{
      text: text,
      type: type,
      title: title,
      time: get_date_time(unix_time)
    }
  end

  defp get_date_time(nil), do: nil

  defp get_date_time(unix_time) do
    {:ok, date_time} = DateTime.from_unix(unix_time)

    DateTime.to_date(date_time)
  end
end
```
Hackernews Part 2
```elixir
defmodule ElixirPopularity.HackernewsApi do
  @moduledoc """
  Fetches data from the Hackernews API, extracts the necessary fields,
  and massages some data
  """

  require Logger

  alias ElixirPopularity.HackernewsItem

  def get_hn_item(resource_id) do
    get_hn_item(resource_id, 4)
  end

  defp get_hn_item(resource_id, retry) do
    response =
      resource_id
      |> gen_api_url()
      |> HTTPoison.get([], hackney: [pool: :hn_id_pool])

    with {_, {:ok, body}} <- {"hn_api", handle_response(response)},
         {_, {:ok, decoded_payload}} <- {"payload_decode", Jason.decode(body)},
         {_, {:ok, data}} <- {"massage_data", massage_data(decoded_payload)} do
      data
    else
      {stage, error} ->
        Logger.warn(
          "Failed attempt #{5 - retry} at stage \"#{stage}\" with Hackernews ID of #{resource_id}. Error details: #{
            inspect(error)
          }"
        )

        if retry > 0 do
          get_hn_item(resource_id, retry - 1)
        else
          Logger.warn("Failed to retrieve Hackernews ID of #{resource_id}.")
          :error
        end
    end
  end

  defp handle_response({:ok, %HTTPoison.Response{status_code: 200, body: body}}) do
    {:ok, body}
  end

  defp handle_response({_, invalid_response}) do
    {:error, invalid_response}
  end

  defp gen_api_url(resource_id) do
    "https://hacker-news.firebaseio.com/v0/item/#{resource_id}.json"
  end

  defp massage_data(data) do
    {:ok,
     HackernewsItem.create(
       data["text"],
       data["type"],
       data["title"],
       data["time"]
     )}
  end
end
```

GenRQM
```elixir
defmodule ElixirPopularity.RMQPublisher do
  @behaviour GenRMQ.Publisher

  @rmq_uri "amqp://rabbitmq:rabbitmq@localhost:5672"
  @hn_exchange "hn_analytics"
  @hn_item_ids_queue "item_ids"
  @hn_bulk_item_data_queue "bulk_item_data"
  @publish_options [persistent: false]

  def init do
    create_rmq_resources()

    [
      uri: @rmq_uri,
      exchange: @hn_exchange
    ]
  end

  def start_link do
    GenRMQ.Publisher.start_link(__MODULE__, name: __MODULE__)
  end

  def hn_id_queue_size do
    GenRMQ.Publisher.message_count(__MODULE__, @hn_item_ids_queue)
  end

  def publish_hn_id(hn_id) do
    GenRMQ.Publisher.publish(__MODULE__, hn_id, @hn_item_ids_queue, @publish_options)
  end

  def publish_hn_items(items) do
    GenRMQ.Publisher.publish(__MODULE__, items, @hn_bulk_item_data_queue, @publish_options)
  end

  def item_id_queue_name, do: @hn_item_ids_queue

  def bulk_item_data_queue_name, do: @hn_bulk_item_data_queue

  defp create_rmq_resources do
    # Setup RabbitMQ connection
    {:ok, connection} = AMQP.Connection.open(@rmq_uri)
    {:ok, channel} = AMQP.Channel.open(connection)

    # Create exchange
    AMQP.Exchange.declare(channel, @hn_exchange, :topic, durable: true)

    # Create queues
    AMQP.Queue.declare(channel, @hn_item_ids_queue, durable: true)
    AMQP.Queue.declare(channel, @hn_bulk_item_data_queue, durable: true)

    # Bind queues to exchange
    AMQP.Queue.bind(channel, @hn_item_ids_queue, @hn_exchange, routing_key: @hn_item_ids_queue)
    AMQP.Queue.bind(channel, @hn_bulk_item_data_queue, @hn_exchange, routing_key: @hn_bulk_item_data_queue)

    # Close the channel as it is no longer needed
    # GenRMQ will manage its own channel
    AMQP.Channel.close(channel)
  end
end
```
## Useful Tools
* GenRMQ
* Grafana
* Prometheus
* Docker

## Comments/ Questions
