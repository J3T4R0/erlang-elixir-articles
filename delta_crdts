## TL;DR
A distributed kv store for Delta CRDTs
### Article Link
https://github.com//derekkraan/delta_crdt_ex
https://github.com/bet365
http://www.erlang-factory.com/static/upload/media/1434558446558020erlanguserconference2015bet365michaelowen.pdf
### Author

## Key Takeaways
* Use riak for operations from basho
* Be smart and use testing

## Useful Code Snippets
```elixir
# start 2 Delta CRDTs
{:ok, crdt1} = DeltaCrdt.start_link(DeltaCrdt.AWLWWMap)
{:ok, crdt2} = DeltaCrdt.start_link(DeltaCrdt.AWLWWMap)

# make them aware of each other
DeltaCrdt.set_neighbours(crdt1, [crdt2])

# show the initial value
DeltaCrdt.read(crdt1)
%{}

# add a key/value in crdt1
DeltaCrdt.mutate(crdt1, :add, ["CRDT", "is magic!"])

# read it after it has been replicated to crdt2
DeltaCrdt.read(crdt2)
%{"CRDT" => "is magic!"}

```

## Useful Tools
* Using webmachine
* Apache Solr

## Comments/ Questions
