riak_core-tutorial
==================

## Multinode Hello World ##

1. Install riak_core rebar_template

    > to może będzie zrobione na maszynce wirtualnej więc na razie nie opisuje
    
2. Generate multinode app  
`./rebar create template=riak_core_multinode appid=sc nodeid=sc`
3. Modify rebar.config to match the relase 1.4.9 and remove:  
   ```
   {deps, [
       {riak_core, "1.4.*",
           {git, "git://github.com/basho/riak_core", {tag, "1.4.9"}}}
   ]}.
   ```

    > At the time of writing this tutorial the latest stable version
    > is 1.4.10. For other version see [releases](https://github.com/basho/riak_core/releases).

4. Generate release with 3 nodes and start them:  
   ```
   make dev rel`  
   for d in dev/dev*; do $d/bin/sc start; done
   ```
5. Check the status of the nodes:  
`./dev/dev1/bin/sc-admin member_status`  
We see that there's only one node in the ring.
6. Join the nodes in a cluster and verify the status again:  
   ```
   for d in dev/dev{2,3}; do $d/bin/sc-admin join sc1@127.0.0.1; done  
   ./dev/dev1/bin/sc-admin member_status  
   ```
   
   Look at the `Ring` column's values. It indicates how much of the key
   space is allocated for a particular node. Over time, each node should
   cover proportional percentage of a ring.
7. Now look how the constant hashing works:
    8. Attach to a node 1:  
       `./dev/dev1/bin/sc attach`  
    9. Run the following snippet to compute the hash on each node:
    ```
    F = fun() ->
        Hashes = [begin
            Node = "sc" ++ integer_to_list(N) ++ "@127.0.0.1",
            rpc:call(list_to_atom(Node), riak_core_util, chash_key, [{<<"ping">>, <<"ping">>}])
        end || N <- [1,2,3]],
        [OneHash] = lists:usort(Hashes)
    end.
    F(), F().
    ```
    10. Note two important things:
        11. Each node returns the same hash for a key.
        12. The hash is the same over time.
13. Now let's see how the objects are dispatch over the ring to vnodes.

    > Jakaś teoria i rysunek obrazujący vnode'a i parametr N.
    >
    > **What is vnode?**
    > A vnode is a virtual node, as opposed to physical node
    > * Each vnode is responsible for one partition on the ring
    > * A vnode is an Erlang process
    > * A vnode is a behavior written atop of the gen_fsm behavior
    > * A vnode handles incoming requests
    > * A vnode potentially stores data to be retrieved later
    > * A vnode is the unit of concurrency, replication, and fault tolerance
    > * Typically many vnodes will run on each physical node
    > 
    > Each machine has a vnode master who's purpose is to keep
    > track of all active vnodes on its node.
    >
    > **What is N?**
    > In a riak_core world it indicates number of vnodes on which we
    > want to perform something. 

    ```
    [Hash] = F().
    riak_core_apl:get_primary_apl(H, _N = 2, sc).
    ```

    `apl` stands for *active preference list*.

## Implementing simple crawler ##

The plan for the next step is to implement a distributed internet
crawler that will be able to download websites and just store them for
later retrieval. The desing is as follows:
* downloading will take place on random vnodes in the cluster;
* a particular vnode will store a content of a given URL;
* an old version of a website will be just replaced by new one.

The picture below shows the system for 3 erlang nodes:
> jakiś rysunek tutaj

### Implementing downloader part  ###

We already have an API for downloading in `sc:download/1` so we only
need to implement a vnode that will be handling the actual download
tasks. A skeleton for the vnode is already there in
`sc_downloader_vnode.erl`. Note, that a vnode have to implement
`riak_core_vnode` behaviour. Here we're focusing on `hadnle_command/3`
callback, that will be invokded by the `riak_core_vnode_master:command/3`.

> More information on the `riak_core_vnode` callbacks can be found
> [here](https://github.com/vitormazzi/try-try-try/tree/master/2011/riak-core-the-vnode#life-cycle-callbacks).

Let's get to coding. First of all, add the asynchronous API to the
vnode:
```erlang
-export([start_vnode/1,
         download/2]).
...
-define(MASTER, sc_downloader_vnode_master).
...

-spec download({chash:index_as_int(), node()}, binary()) -> term().
download(IdxNode, URL) ->
    riak_core_vnode_master:command(IdxNode, {download, URL}, ?MASTER).
```

> `MASTER` indicates the ID of the master vnode for downloader vonodes.

Next, implement the command:
```erlang
...
handle_command({download, URL} = Req, _Sender, State) ->
    print_request_info(State#state.partition, node(), Req),
    try
        Content = download(URL),
        store(URL, Content)
    catch
        throw:{download_error, Reason} ->
            ?PRINT({request_failed, Req, Reason})
    end,
    {noreply, State};
...
```

In the final step uncomment specification for
`sc_downloader_vnode_master` in `sc_sup.erl` and add it to
the supervisor's child list. Additionaly, register the vnode in
`sc_app.erl`:
```erlang
...
    ok = riak_core:register([{vnode_module, sc_vnode}]),
    ok = riak_core:register([{vnode_module, sc_downloader_vnode}]),
    ok = riak_core_ring_events:add_guarded_handler(sc_ring_event_handler, []),
....
```

To test our new funcionality stop the whole cluster, clean project,
build devrel again and form the cluster:
```bash
for d in dev/dev*; do $d/bin/sc stop; done
make devclean && make devrel
for d in dev/dev*; do $d/bin/sc start; done
for d in dev/dev{2,3}; do $d/bin/sc-admin join sc1@127.0.0.1; done
```
Once we have all the setup up and running attach to one of the nodes
and observe the logs of the other two nodes. Assuming that you attached
to dev1:
```bash
tail -f dev/dev2/log/erlang.log.1
tail -f dev/dev3/log/erlang.log.1
```

Experiment a bit with `sc:download/1` API:
`1> sc:download(<<"http://www.erlang.org">>).` and note that the
reuqests are serverd by a random partitions on different nodes.
Effectively it means request hit different vnodes (a vnode is
responsible for one partition).

> "The randomness" is achieved by picking a vnode for random document
>  index. See `sc:get_random_dument_index/0`.

### Implementing storage part ###

Let's move to our storage system. As above, the API is already
implemented in `sc:store/2` and `sc:get_content/2` (just uncomment
all the lines these functions). Recall, that in this case the same
vnode will be chosen for storing or retireving data for a particular URL.

Similarily as in the previous example we need a vnode to do our job.
There is already such a vnode implemented `sc_storage_vnode.erl`.
Please, have look at its `get_content/3` API function. It invokes
the command using `riak_core_vnode_master:sync_spawn_command/3` that
is asynchronus but doesn't block the vnode. The difference is also
in the command for retrieving the content as we return a reply.

To get it working you also have to take care of the master vnode for
storage in `sc_sup.erl` and register the `sc_storage_vnode.erl` -
analogously as with downloader vnode.

When you're done restart the whole machinery using the snippet above.
Next, let's beging testing how it works: attach to one node and "tailf"
the logs of the others. Download your favourie website and get
its content:
```
sc:download(<<"http://joemonster.org/">>).
sc:download(<<"http://joemonster.org/">>).
sc:get_content(<<"http://joemonster.org/">>).
```

You would expect that *download* requests will be served by different
vnodes and each *store* and *get_content* requests by the same vnode.
But, hey! what if `get_content/1` returns an empty list even though
the request match the right partition?! Well, it's possible...

The explanation behind this behaviour is that, when you start your
first node it servers all the partitions which in practice means that
it runs all the vnodes of each kind (by default 64 partitions are
created). When new nodes join the cluster the partitions are spread
across them but it happens in the background - strictly speaking: while
the cluster is serving reuquest it's moving vnodes to other physical
nodes in the same time. But riak_core have no idea how to move our
data so it's just lost! Terrible ha?

To observe the whole system working as expected you need to wait for
the cluster to come into "stable state". Just check the status:
`./dev/dev1/bin/sc-admin member_status`
When there're no pending changes it means that no partitions will be
moved. Now you can experiment again and make sure, that requests are
served by appropriate partitions, vnodes and nodes.

In the next part I'm going to explain how to prepare for moving
a vnode: so called *handoff*.

## Handoff ##

### What is handoff?  ###

A *handoff* occurss when a vnode realizes that it's not on the proper
physical node. Such a situation can take place when:
* a node is added or removed
* a node comes alive after it has been down.
In riak_core there's a periodic "home check* that verifies whether
a vnode uses correct physical node. If that's not true for some vnode
it will go into handoff mode and data will be transferred. 

### How to handle handoff? ###

When riak_core decides to perform a handoff it calls two functions:
`Mod:handoff_starting/2` and `Mod:is_empty/1`. Through the first one
a vnode can agreeon or not to proceede with the handoff. The first one
indicates if there's any data to be transfered and it's interesting in
our case. So lets code it in `sc_storage_vnode.erl`:
```erlang
is_empty(State) ->
    case dict:size(State#state.store) of
        0 ->
            {true, State};
        _ ->
            {false, State}
    end.
```

When the framework decides to start handoff it sends `?FOLD_REQ` that is
a request representing how to fold over the data held by the vnode.
This request is supposed to be handled in `Mod:handle_handoff_command/3`
and contains "folding function" along with initial accumulator.  Let's
add it to our storage vnode:
```erlang
handle_handoff_command(?FOLD_REQ{foldfun = Fun, acc0=Acc0},
                       _Sender, State) ->
    Acc = dict:fold(Fun, Acc0, State#state.store),
    {reply, Acc, State}.
```

> If you're like my and this ?FOLD_REQ looks strange to you have a look
> at the [riak_core_vnode.hrl](https://github.com/basho/riak_core/blob/1.0/include/riak_core_vnode.hrl)
> that reveals that the macro is just a record.


So, what's next with this magic handoff? Well, at this point things
are simple: each iteration of "folding function" calls
`Mod:encode_handoff_item/2` that just do what it is supposed to do:
encode data before sending it to the targe vnode. The target vnode,
suprisingly (!), decodes the data in `Mod:handle_handoff_data`. In
this tutorial we are using extremely complex method of encoding so
write the following code in your vnode really careful:
```erlang
encode_handoff_item(URL, Content) ->
    term_to_binary({URL, Content}).
...
handle_handoff_data(Data, State) ->
    {URL, Content} = binary_to_term(Data),
    Dict = dict:store(URL, Content, State#state.store),
    {reply, ok, State#state{store = Dict}}.
```

And that's it. Our system is now "dead node resistant". One more thing
is worth noting here: after the handoff is completed `Mod:delete/1`
is called and the vnode will be terminated just after this call.
During the termination `Mod:terminate/2` will be called too.

> I did not present all the callbacks related to handoff. For more
> information go to great explanation
[here](https://github.com/vitormazzi/try-try-try/tree/master/2011/riak-core-the-vnode#handoff).
> If you need more details look at the
[basho wiki](https://github.com/basho/riak_core/wiki/Handoffs).

### See handoff in action ###

Now, when we have handoff impleted, build devrel, start the cluster,
**but only join dev2 to dev1**. We want to observe how the partitions
are moved:
```bash
for d in dev/dev*; do $d/bin/sc start; done
dev/dev2/bin/sc-admin join sc1@127.0.0.1
```
Wait for the ring to get populated symmetricaly across the nodes. Use
`./dev/dev1/bin/sc-admin member_status` to check the status.

Next attach to the console of node from the cluster and look at the
logs of the other node. Download some websites and write down on
which erlang node they are stored. Next join the 3rd node to the cluster
and try to retrieve content from previously downloaded sites. You
should see that some of them will be serverd from the new node
and it's transparent to a user.

### Notatki ###
1. Erlang R15B03 jest potrzebny
   

