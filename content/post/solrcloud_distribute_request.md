---
title: "SolrCloud Handle Distributed Request And Load Balance"
date: 2019-06-21T11:40:16+08:00
categories:
- solr
tags:
- solr
- lucene
- SolrCloud
keywords:
- SolrCloud
- Distributed
- Load
- Balance
#thumbnailImage: //example.com/image.jpg
---

This note is based on source code reading.

As in SolrCloud, the query request is sent to one node. SolrCloud would automatically distribute the query to each shards. To have better performance, It enable load balance to make the distributed request evenly sent to each node. 
<!--more-->

For a collection, solr core(replica)’s `SearchHandler` will handle the query request through function `handleRequestBody`.

```java
public void handleRequestBody(SolrQueryRequest req, SolrQueryResponse rsp) throws Exception
  {
    ...
    ResponseBuilder rb = new ResponseBuilder(req, rsp, components);
    ...
    final ShardHandler shardHandler1 = getAndPrepShardHandler(req, rb); // creates a ShardHandler object only if it's needed
    ...
    if (!rb.isDistrib) {
      // a normal non-distributed request
	  ...
    } else {
      // a distributed request
      ...
    }
    ...
  }
```

For a query request, there are several parameters will affect whether the request is distributed or which shards will be distributed to.

|     Parameter       | Value                   | Description                                                                                                                     |
|:-----------------:  |------------------------ |-------------------------------------------------------------------------------------------------------------------------------  |
| shards              | Shard ids, or replicas  | Search specify shards or replicas only. If this parameter is not provided, which is default, SolrCloud will query all shards. [Details](https://lucene.apache.org/solr/guide/6_6/distributed-requests.html#DistributedRequests-LimitingWhichShardsareQueried )   |
| distrib             | true/false              | If the value is false, only execute query on the replica/core which receive the request.                                        |
| preferLocalShards   | true/false              | if this set to true, it'll query on local replicas. [Details](https://lucene.apache.org/solr/guide/6_6/distributed-requests.html#DistributedRequests-PreferLocalShards )                                                                             |


Let’s have a look for the distributed request and focus on the load balance.
Once the function handleRequestBody receive a request. It inits a ResponseBuilder. And then let getAndPrepShardHandler function to get and prepare a ShardHandler which used to send distributed request.
`SearchHandler.handleRequestBody() ----> SearchHandler.getAndPreShardHandler().`

```java
private ShardHandler getAndPrepShardHandler(SolrQueryRequest req, ResponseBuilder rb) {
    ShardHandler shardHandler = null;

    CoreContainer cc = req.getCore().getCoreContainer();
    boolean isZkAware = cc.isZooKeeperAware();
    rb.isDistrib = req.getParams().getBool(DISTRIB, isZkAware);
    if (!rb.isDistrib) {
      // for back compat, a shards param with URLs like localhost:8983/solr will mean that this
      // search is distributed.
      final String shards = req.getParams().get(ShardParams.SHARDS);
      rb.isDistrib = ((shards != null) && (shards.indexOf('/') > 0));
    }
  
    if (rb.isDistrib) {
      shardHandler = shardHandlerFactory.getShardHandler();
      shardHandler.prepDistributed(rb);
      if (!rb.isDistrib) {
        shardHandler = null; // request is not distributed after all and so the shard handler is not needed
      }
    }

    if(isZkAware) {
      ZkController zkController = cc.getZkController();
      NamedList<Object> headers = rb.rsp.getResponseHeader();
      if(headers != null) {
        headers.add("zkConnected", 
            zkController != null 
          ? !zkController.getZkClient().getConnectionManager().isLikelyExpired() 
          : false);
      }
    
    }

    return shardHandler;
  }
```

The main idea for this function is to check whether current request is distributed request. 
And prepare the shards info for current collection. Store all needed info into ResponseBuilder.
At the begin, get the CoreContainer and check whether zookeeper exists.

If zookeeper exists, means current env is SolrCloud not a single solr env. Then, if parameter “distrib” is not false, current request will treat as distributed request.
Else, current request will treat as non-distributed request.
If request is distributed, get shardHandler and execute shardHandler.preDistributed(rb) to prepare shards info.
Last, if zookeeper exists, set zookeeper info into ResponseBuilder’s response header.

Let’s have a look how shardHandler prepare shards info.
Explain two concept here.

| Parameter   |                            Description                  |
|-----------  |---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| rb.slices   | Slice is a logical shard for code area. rb.slices contains the shard ID. like “shard1, shard2”                                                                                                      |
| rb.shards   | Shard here is a list of replicas. The format is like "http://node:port/solr/collection_shard0_replica1\|http://node:port/solr/collection_shard0_replica2”. This actually for load balance purposes.  |

`SearchHandler.handleRequestBody() ----> SearchHandler.getAndPreShardHandler() ----HttpShardHandler.preDistributed()`

For this function, it can be separated into two part. First to get Slices info and init slice number of shards.

```java
public void prepDistributed(ResponseBuilder rb) {
  final SolrQueryRequest req = rb.req;
  final SolrParams params = req.getParams();
  final String shards = params.get(ShardParams.SHARDS);
  ...
  final ReplicaListTransformer replicaListTransformer = httpShardHandlerFactory.getReplicaListTransformer(req);
  if (shards != null) {
    List<String> lst = StrUtils.splitSmart(shards, ",", true);
    rb.shards = lst.toArray(new String[lst.size()]);
    rb.slices = new String[rb.shards.length];
    if (zkController != null) {
      // figure out which shards are slices
      for (int i=0; i<rb.shards.length; i++) {
        if (rb.shards[i].indexOf('/') < 0) {
          // this is a logical shard
          rb.slices[i] = rb.shards[i];
          rb.shards[i] = null;
        }
      }
    }
  } else if (zkController != null) {
    // we weren't provided with an explicit list of slices to query via "shards", so use the cluster state
    ...
    // we need to find out what collections this request is for.
    // A comma-separated list of specified collections.
    // Eg: "collection1,collection2,collection3"
    ...
    // Store the logical slices in the ResponseBuilder and create a new
    // String array to hold the physical shards (which will be mapped
    // later).
    rb.slices = slices.keySet().toArray(new String[slices.size()]);
    rb.shards = new String[rb.slices.length];
  }
  //
  // Map slices to shards
  //
...
...
}
```

Before get slice info, get a `ReplicaListTransformer`, it’s used to transforms the passed in list of request replicas. Random reorder the list for load balance request. If `preferLocalReplicas` set to “true”. It’ll sort replicas by comparing the node host address. The local replicas will be placed from the beginning of the replica list.

`SearchHandler.handleRequestBody() ----> SearchHandler.getAndPreShardHandler() ----HttpShardHandler.preDistributed() ----> HttpShardHandlerFactory.getReplicaListTransformer()`

```java
protected ReplicaListTransformer getReplicaListTransformer(final SolrQueryRequest req)
{
  final SolrParams params = req.getParams();
  if (params.getBool(CommonParams.PREFER_LOCAL_SHARDS, false)) {
    final CoreDescriptor coreDescriptor = req.getCore().getCoreDescriptor();
    final ZkController zkController = req.getCore().getCoreContainer().getZkController();
    final String preferredHostAddress = (zkController != null) ? zkController.getBaseUrl() : null;
    if (preferredHostAddress == null) {
      log.warn("Couldn't determine current host address to prefer local shards");
    } else {
      return new ShufflingReplicaListTransformer(r) {
        @Override
        public void transform(List<?> choices)
        {
          if (choices.size() > 1) {
            super.transform(choices);
            if (log.isDebugEnabled()) {
              log.debug("Trying to prefer local shard on {} among the choices: {}",
                  preferredHostAddress, Arrays.toString(choices.toArray()));
            }
            choices.sort(new IsOnPreferredHostComparator(preferredHostAddress));
            if (log.isDebugEnabled()) {
              log.debug("Applied local shard preference for choices: {}",
                  Arrays.toString(choices.toArray()));
            }
          }
        }
      };
    }
  }
  return shufflingReplicaListTransformer;
}
```
If the request parameters already exists “shards”.
Value is shard ids, set them in rb.slices. Init rb.shards with empty but same size with rb.slices. Then in future, get replicas for each shard and set in rb.shards(for distribute request, with load balance cause it’ll random pick a replica later).

Value is shard urls that separate by “,”, set them in rb.shards(for distribute request, no load balance cause no replicas selected for further load balance request).
Value is shard’s replicas url that seperate by “|”, set them in rb.shards(for distribute request, with load balance cause it’ll random pick a replica later).

If no “shards” parameter provided and solr connect to zookeeper(which means in SolrCloud mode).
Get collection or collections(able to query multiple collections by setting “collection” parameter) shards info from zookeeper and set into rb.slices. Init rb.shards with empty but same size with rb.slices. Then in future, get replicas for each shard and set in rb.shards(for distribute request, with load balance cause it’ll random pick a replica later).

Then the second part is mapping slices to shards.

```java
public void prepDistributed(ResponseBuilder rb) {
  // Get slices(logic shard object) info and init slice number of shards
  ...
  //
  // Map slices to shards
  //
  if (zkController != null) {
    // Are we hosting the shard that this request is for, and are we active? If so, then handle it ourselves
    // and make it a non-distributed request.
    String ourSlice = cloudDescriptor.getShardId();
    String ourCollection = cloudDescriptor.getCollectionName();
    // Some requests may only be fulfilled by replicas of type Replica.Type.NRT
    boolean onlyNrtReplicas = Boolean.TRUE == req.getContext().get(ONLY_NRT_REPLICAS);
    if (rb.slices.length == 1 && rb.slices[0] != null
        && ( rb.slices[0].equals(ourSlice) || rb.slices[0].equals(ourCollection + "_" + ourSlice) )  // handle the <collection>_<slice> format
        && cloudDescriptor.getLastPublished() == Replica.State.ACTIVE
        && (!onlyNrtReplicas || cloudDescriptor.getReplicaType() == Replica.Type.NRT)) {
      ...
      if (shortCircuit) {
        rb.isDistrib = false;
        rb.shortCircuitedURL = ZkCoreNodeProps.getCoreUrl(zkController.getBaseUrl(), coreDescriptor.getName());
        return;
      }
      // We shouldn't need to do anything to handle "shard.rows" since it was previously meant to be an optimization?
    }
    ...
  }
  ...
}
```

To map slices to shards, current core/replica should have zookeeper connection(which mean SolrCloud mode). Here’s a simple optimize, if the rb.slices only have one, and it’s same shard with the replica/core which receive current request. Then mark the request non-distributed just handle it by itself. Finish the prepare and return.

If have more than one rb.slices, continue map slices to shards.
```java
public void prepDistributed(ResponseBuilder rb) {
  // Get slices(logic shard object) info and init slice number of shards
  ...
  // Map slices to shards
  if (zkController != null) {
    // Are we hosting the shard that this request is for, and are we active? If so, then handle it ourselves and make it a non-distributed request.
    ...
    for (int i=0; i<rb.shards.length; i++) {
      if (rb.shards[i] != null) {
        final List<String> shardUrls = StrUtils.splitSmart(rb.shards[i], "|", true);
        replicaListTransformer.transform(shardUrls);
        // And now recreate the | delimited list of equivalent servers
        rb.shards[i] = createSliceShardsStr(shardUrls);
      } else {
        ...
        final List<Replica> eligibleSliceReplicas = collectEligibleReplicas(slice, clusterState, onlyNrtReplicas, isShardLeader);
        final List<String> shardUrls = transformReplicasToShardUrls(replicaListTransformer, eligibleSliceReplicas);
        // And now recreate the | delimited list of equivalent servers
        final String sliceShardsStr = createSliceShardsStr(shardUrls);
        ...
        rb.shards[i] = sliceShardsStr;
      }
    }
  }
  ...
}
```

For each shard in `rb.shards`, if value is not null.
Using `ReplicaListTransformer` to reorder replicas. The transform function we already discussed before. Recreate the replicas back to rb.shards. Now the order of replicas is different from original value for each shard in rb.shards.

If value is null.
No replicas info for each shard, so get shard active replicas. Run transformReplicasToShardUrls to reorder active replicas and get replcia urls for load balance.

`SearchHandler.handleRequestBody() ----> SearchHandler.getAndPreShardHandler() ----HttpShardHandler.preDistributed() ----> ----HttpShardHandler.transformReplicasToShardUrls()`

```java
private static List<String> transformReplicasToShardUrls(final ReplicaListTransformer replicaListTransformer, final List<Replica> eligibleSliceReplicas) {
  replicaListTransformer.transform(eligibleSliceReplicas);
  final List<String> shardUrls = new ArrayList<>(eligibleSliceReplicas.size());
  for (Replica replica : eligibleSliceReplicas) {
    String url = ZkCoreNodeProps.getCoreUrl(replica);
    shardUrls.add(url);
  }
  return shardUrls;
}
```

After set all needed info to `ResponseBuilder(rb here)`, the distributed prepare process finished.

Let’s go back to `SearchHandler.handleRequestBody()`, 

```java
public void handleRequestBody(SolrQueryRequest req, SolrQueryResponse rsp) throws Exception
  {
    ...
    if (!rb.isDistrib) {
      // a normal non-distributed request
	  ...
    } else {
      // a distributed request
      ...
      int nextStage = 0;
      do {
        rb.stage = nextStage;
        nextStage = ResponseBuilder.STAGE_DONE;
        // call all components
        for( SearchComponent c : components ) {
          // the next stage is the minimum of what all components report
          nextStage = Math.min(nextStage, c.distributedProcess(rb));
        }
        // check the outgoing queue and send requests
        while (rb.outgoing.size() > 0) {
          // submit all current request tasks at once
          while (rb.outgoing.size() > 0) {
            ...
            // TODO: map from shard to address[]
            for (String shard : sreq.actualShards) {
              ...
              shardHandler1.submit(sreq, shard, params);
            }
          }
          // now wait for replies, but if anyone puts more requests on
          // the outgoing queue, send them out immediately (by exiting
          // this loop)
          boolean tolerant = rb.req.getParams().getBool(ShardParams.SHARDS_TOLERANT, false);
          while (rb.outgoing.size() == 0) {
            ...
            // let the components see the responses to the request
            for(SearchComponent c : components) {
              c.handleResponses(rb, srsp.getShardRequest());
            }
          }
        }
        for(SearchComponent c : components) {
          c.finishStage(rb);
        }
        // we are done when the next stage is MAX_VALUE
      } while (nextStage != Integer.MAX_VALUE);
    }
    ...
  }
```

To process a request, solr separates the whole process into several stages. For each stage and each component, it’ll execute below steps:
Get next process stage, the next stage is the minimum of what all components report.
For current stage, if components need to distributed execution on each shard, update rb.outgoing queue. The job is done through each components by `c.distributedProcess()`.

If `rb.outgoing` queue size > 0, send distributed requests by executing shardHandler1.submit() for each shard. Wait response for each request and build response.
If there’s no next stage, finish distributed request.

Let’s take a look for a example, a component need execute on each shard, QueryComponent.
During c.distributedProcess(), it’ll add ShardRequest into rb.outgoing queue.
`SearchHandler.handleRequestBody() ----> QueryComponent.distributedProcess()`
```java
public int distributedProcess(ResponseBuilder rb) throws IOException {
  if (rb.grouping()) {
    return groupedDistributedProcess(rb);
  } else {
    return regularDistributedProcess(rb);
  }
}
```

`SearchHandler.handleRequestBody() ----> QueryComponent.distributedProcess() ----> QueryComponent.regularDistributedProcess()`

```java
protected int regularDistributedProcess(ResponseBuilder rb) {
  if (rb.stage < ResponseBuilder.STAGE_PARSE_QUERY)
    return ResponseBuilder.STAGE_PARSE_QUERY;
  if (rb.stage == ResponseBuilder.STAGE_PARSE_QUERY) {
    createDistributedStats(rb);
    return ResponseBuilder.STAGE_EXECUTE_QUERY;
  }
  if (rb.stage < ResponseBuilder.STAGE_EXECUTE_QUERY) return ResponseBuilder.STAGE_EXECUTE_QUERY;
  if (rb.stage == ResponseBuilder.STAGE_EXECUTE_QUERY) {
    createMainQuery(rb);
    return ResponseBuilder.STAGE_GET_FIELDS;
  }
  if (rb.stage < ResponseBuilder.STAGE_GET_FIELDS) return ResponseBuilder.STAGE_GET_FIELDS;
  if (rb.stage == ResponseBuilder.STAGE_GET_FIELDS && !rb.onePassDistributedQuery) {
    createRetrieveDocs(rb);
    return ResponseBuilder.STAGE_DONE;
  }
  return ResponseBuilder.STAGE_DONE;
}
```

If the state now is “STAGE_EXECUTE_QUERY”, run `createMainQuery(rb)`.

`SearchHandler.handleRequestBody() ----> QueryComponent.distributedProcess() ----> QueryComponent.regularDistributedProcess() ----> QueryComponent.createMainQuery()`

```java
protected void createMainQuery(ResponseBuilder rb) {
  ShardRequest sreq = new ShardRequest();
  sreq.purpose = ShardRequest.PURPOSE_GET_TOP_IDS  ...
  sreq.params = new ModifiableSolrParams(rb.req.getParams());
  ...
  // Set parameters for ShardRequest.
  ...
  rb.addRequest(this, sreq);
}
```

The main task is to set query parameters for `ShardRequest`, and add the `ShardRequest` into `ResponseBuilder`.

`SearchHandler.handleRequestBody() ----> QueryComponent.distributedProcess() ----> QueryComponent.regularDistributedProcess() ----> QueryComponent.createMainQuery() ----> ResponseBuilder.addRequest()`

```java
public void addRequest(SearchComponent me, ShardRequest sreq) {
  outgoing.add(sreq);
  if ((sreq.purpose & ShardRequest.PURPOSE_PRIVATE) == 0) {
    // if this isn't a private request, let other components modify it.
    for (SearchComponent component : components) {
      if (component != me) {
        component.modifyRequest(this, me, sreq);
      }
    }
  }
}
Actually, the ShardRequest added into ResponseBuilder.outgoing queue.

Go back to SearchHandler.handleRequestBody(). We know that if the ResponseBuilder.outgoing size is > 0, It’ll submit the ShardRequest for each shard.
SearchHandler.handleRequestBody() ---->shardhandler1(HttpShardhandler).submit()
public void submit(final ShardRequest sreq, final String shard, final ModifiableSolrParams params) {
  // do this outside of the callable for thread safety reasons
  final List<String> urls = getURLs(shard);
  Callable<ShardResponse> task = () -> {
    ShardResponse srsp = new ShardResponse();
    ...
    try {
      ...
      if (urls.size() <= 1) {
        String url = urls.get(0);
        srsp.setShardAddress(url);
        try (SolrClient client = new Builder(url).withHttpClient(httpClient).build()) {
          ssr.nl = client.request(req);
        }
      } else {
        LBHttpSolrClient.Rsp rsp = httpShardHandlerFactory.makeLoadBalancedRequest(req, urls);
        ssr.nl = rsp.getResponse();
        srsp.setShardAddress(rsp.getServer());
      }
    }
    catch( ConnectException cex ) {
      ...
    }
    ssr.elapsedTime = TimeUnit.MILLISECONDS.convert(System.nanoTime() - startTime, TimeUnit.NANOSECONDS);
    return transfomResponse(sreq, srsp, shard);
  };
  try {
    ...
    pending.add( completionService.submit(task) );
  } finally {
    ...
  }
}
```

The value of argument “shard” is look like “http://node:port/solr/collection_shard0_replica1|http://node:port/solr/collection_shard0_replica2”
At beginning, split argument “shard” into url list by “|”.
Here create a Callable object called task. It wrap request execution.

If url list size is less than 2, send client request to the url.
Else, make load balance request.
Then the Callable object task add into pending which is a Future set. This will perform async request and help get results by Future in thread.

For load balance request:
`SearchHandler.handleRequestBody() ---->shardhandler1(HttpShardhandler).submit() ---->HttpShardHandlerFactory.makeLoadBalanceRequest()`
```java
public LBHttpSolrClient.Rsp makeLoadBalancedRequest(final QueryRequest req, List<String> urls)
  throws SolrServerException, IOException {
  return loadbalancer.request(newLBHttpSolrClientReq(req, urls));
}
```

The `loadbalance.request` tries to query a live replica url from the list provided in Req. Replicas in the dead pool are skipped. If a request fails due to an IOException, the replica is moved to the dead pool for a certain period of time, or until a test request on that replica succeeds. 

Replicas are queried in the exact order given(except replicas currently in the dead pool are skipped), which get reordered by `replicaListTransformer.transform()` at `SearchHandler.handleRequestBody() ----> SearchHandler.getAndPreShardHandler() ----HttpShardHandler.preDistributed()`.

If no live replicas from the provided list remain to be tried, a number of previously skipped dead replicas will be tried. `Req.getNumDeadServersToTry()` controls how many dead replicas will be tried. If no live replicas are found a `SolrServerException` is thrown.

