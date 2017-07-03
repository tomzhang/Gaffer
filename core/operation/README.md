Copyright 2017 Crown Copyright

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

This page has been copied from the Operation module README. To make any changes please update that README and this page will be automatically updated when the next release is done.

Operations
============

This module contains the `Operation` interfaces and core operation implementations.

It is assumed that all Gaffer graphs will be able to handle these core operations.

An `Operation` implementation defines an operation to be processed on a
graph, or on a set of results which are returned by another operation. An
`Operation` class contains the configuration required to tell Gaffer how
to carry out the operation. For example, the `AddElements` operation contains
the elements to be added. The `GetElements` operation contains the seeds
to use to find elements in the graph and the filters to apply to the query.
The operation classes themselves should not contain the logic to carry out
the operation (as this may vary between the different supported store types),
just the configuration.

For each operation, each Gaffer store will have an `OperationHandler`, where
the processing logic is contained. This enables operations to be handled
differently in each store.

Operations can be chained together to form an `OperationChain`. When an
operation chain is executed on a Gaffer graph the output of one operation
is passed to the input of the next.

An `OperationChain.Builder` is provided to help with constructing a valid
operation chain - it ensures the output type of an operation matches the
input type of the next.

## How to write an Operation

Operations should be written to be as generic as possible to allow them
to be applied to different graphs/stores.

Operations must be JSON serialisable in order to be used via the REST API
- i.e. there must be a public constructor and all the fields should have
getters and setters.

Operation implementations need to implement the `Operation` interface and
the extra interfaces they they wish to make use of. For example an operation
that takes a single input value should implement the `Input` interface.

Here is a list of some of the common interfaces:
- uk.gov.gchq.gaffer.operation.io.Input
- uk.gov.gchq.gaffer.operation.io.Output
- uk.gov.gchq.gaffer.operation.io.InputOutput - Use this instead of Input
and Output if your operation takes both input and output.
- uk.gov.gchq.gaffer.operation.io.MultiInput - Use this in addition if you
operation takes multiple inputs. This will help with JSON serialisation
- uk.gov.gchq.gaffer.operation.SeedMatching
- uk.gov.gchq.gaffer.operation.Validatable
- uk.gov.gchq.gaffer.operation.graph.OperationView
- uk.gov.gchq.gaffer.operation.graph.GraphFilters
- uk.gov.gchq.gaffer.operation.graph.SeededGraphFilters
- uk.gov.gchq.gaffer.operation.Options

Each operation implementation should have a corresponding unit test class
that extends the `OperationTest` class.

Operation implementations should override the close method and ensure all
closeable fields are closed.

Any fields that are required should be annotated with the Required annotation.

All implementations should also have a static inner `Builder` class that
implements the required builders. For example:

```java
public static class Builder extends Operation.BaseBuilder<GetElements, Builder>
        implements InputOutput.Builder<GetElements, Iterable<? extends ElementId>, CloseableIterable<? extends Element>, Builder>,
        MultiInput.Builder<GetElements, ElementId, Builder>,
        SeededGraphFilters.Builder<GetElements, Builder>,
        SeedMatching.Builder<GetElements, Builder>,
        Options.Builder<GetElements, Builder> {
    public Builder() {
            super(new GetElements());
    }
}
```

## FAQs
Here are some frequently asked questions.

#### If a do a query like GetElements or GetAdjacentIds the response type is a CloseableIterable - why?
To avoid loading all the results into memory, Gaffer stores should return an iterable that lazily loads and returns the data as a user iterates around the results. In the cases of Accumulo and HBase this means a connection to Accumulo/HBase must remain open whilst you iterate around the results. This closeable iterable should automatically close itself when you get to the end of the results. However, if you decide not to read all the results, i.e you just want to check if the results are not empty !results.iterator().hasNext() or an exception is thrown whilst iterating around the results, then the results iterable will not be closed and hence the connection to Accumulo/HBase will remain open. Therefore, to be safe you should always consume the results in a try-with-resources block.

#### Following on from the previous question, why can't I iterate around the results in parallel?
As mentioned above the results iterable holds a connection open to Accumulo/HBase. To avoid opening multiple connections accidentally leaving the connections open, the Accumulo and HBase stores only allow one iterator to be active at a time. When you call .iterator() the connection is opened. If you call .iterator() again, the original connection is closed and a new connection is opened. This means you can't process the iterable in parallel using Java 8's streaming api. If the results will fit in memory you could add them to a Set/List and then process that collection in parallel.

#### How do I return all my results summarised?
You need to provide a View to override the groupBy fields for all the element groups defined in the Schema. If you set the groupBy field to an empty array it will mean no properties will be included in the element key, i.e all the properties will be summarised. You can do this be provided a View like this:

```json
"view": {
    "globalElements" : [{
        "groupBy" : []
    }]
}
```

#### My queries are returning duplicate results - why and how can I deduplicate them?
For example, if you have a Graph containing the Edge A-B and you do a GetElements with a large number of seeds, with the first seed A and the last seed B, then you will get the Edge A-B back twice. This is because Gaffer stores lazily return the results for your query to avoid loading all the results into memory so it will not realise the A-B has been queried for twice.

You can deduplicate your results in memory using the [ToSet](https://github.com/gchq/Gaffer/wiki/Operation-examples#toset-example) operation. But, be careful to only use this when you have a small number of results. It might be worth also using the [Limit](https://github.com/gchq/Gaffer/wiki/Operation-examples#limit-example) operation prior to ToSet to ensure you don't run out of memory.

e.g: 

```java
new OperationChain.Builder()
    .first(new GetAllElements())
    .then(new Limit<>(1000000))
    .then(new ToSet<>())
    .build();
```

#### I have just done a GetElements and now I want to do a second hop around the graph, but when I do a GetElements followed by another GetElements I get strange results.
You can seed a get related elements operation with vertices (EntityIds) or edges (EdgeIds). If you seed the operation with edges you will get back the Entities at the source and destination of the provided edges, in addition to the edges that match your seed.

For example, using this graph:

```

    --> 4 <--
  /     ^     \
 /      |      \
1  -->  2  -->  3
         \
           -->  5
```

If you start with seed 1 and do a GetElements (related) then you would get back:

```
1
2
4
1 -> 2
1 -> 4
``` 

If you chain this into another GetElements then you would get back some strange results:

```
#Seed 1 causes these results
1
2
1 -> 2
1 -> 4

#Seed 2 causes these results
2
1
3
4
5
1 -> 2
2 -> 3
2 -> 4
2 -> 5

#Seed 4 causes these results
4
1
2
3
1 -> 4
2 -> 4
3 -> 4

#Seed 1 -> 2 causes these results
1
2
1 -> 2

#Seed 1 -> 4 causes these results
1
4
1 -> 4
```

So you get a lot of duplicates and unwanted results. What you really want to do is to use the GetAdjacentIds query to simply hop down the first edges and return just the vertices at the opposite end of the related edges. You can still provide a View and apply filters to the edges you traverse down. In addition it is useful to add a direction to the query so you don't go back down the original edges. So let's say we only want to traverse down outgoing edges. Doing a GetAdjacentIds with seed 1 would return:

```
2
4
``` 

Then you can do another GetAdjacentIds and get the following:

```
#Seed 2 causes these results
4
5

#Seed 4 does not have any outgoing edges so it doesn't match anything
```

You can continue doing multiple GetAdjacentIds to traverse around the Graph further. If you want the properties on the edges to be returned you can use GetElements in your final operation in your chain.