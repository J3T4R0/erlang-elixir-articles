## TL;DR
Riak Core on Partisan KV storage 
### Article Link
http://marianoguerra.org/posts/riak-core-on-partisan-on-elixir-tutorial-getting-started.html
### Author
Mariano Guerra
## Key Takeaways
* Civile DB 
* Node Session Hand offs

## Useful Code Snippets
Application
```elixir
defmodule Civile do
  use Application
  require Logger

  def start(_type, _args) do
    case Civile.Supervisor.start_link do
      {:ok, pid} ->
        {:ok, pid}
      {:error, reason} ->
        Logger.error("Unable to start Civile supervisor because: #{inspect reason}")
    end
  end
end

```
Civile_service
 ```elixir
 defmodule Civile.Service do
  def ping(v \\ 1) do
    send_cmd("ping#{v}", {:ping, v})
  end

  def put(k, v) do
    send_cmd(k, {:put, {k, v}})
  end

  def get(k) do
    send_cmd(k, {:get, k})
  end

  defp send_cmd(k, cmd) do
    idx = :riak_core_util.chash_key({"civile", k})
    pref_list = :riak_core_apl.get_primary_apl(idx, 1, Civile.Service)

    [{index_node, _type}] = pref_list

    :riak_core_vnode_master.sync_command(index_node, cmd, Civile.VNode_master)
  end
end
```
Get
```elixir
def handle_command({:get, k}, _sender, state = %{table_id: table_id, partition: partition}) do
  res =
    case :ets.lookup(table_id, k) do
      [] ->
        {:ok, node(), partition, nil}

      [{_, value}] ->
        {:ok, node(), partition, value}
    end

  {:reply, res, state}
end
```
Put
```elixir
def handle_command({:put, {k, v}}, _sender, state = %{table_id: table_id, partition: partition}) do
  :ets.insert(table_id, {k, v})
  res = {:ok, node(), partition, nil}
  {:reply, res, state}
end
Init
```elixir
def init([partition]) do
  table_name = :erlang.list_to_atom('civile_' ++ :erlang.integer_to_list(partition))

  table_id =
    :ets.new(table_name, [:set, {:write_concurrency, false}, {:read_concurrency, false}])

  state = %{
    partition: partition,
    table_name: table_name,
    table_id: table_id
  }

  {:ok, state}
end
```


## Useful Tools
* Ets data storage
* LASP riak_core leveled kafka 

## Comments/ Questions


