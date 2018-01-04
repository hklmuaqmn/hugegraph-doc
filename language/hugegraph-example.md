# HugeGraph Examples

## 1. 概述

本示例将[Titandb Getting Started](http://s3.thinkaurelius.com/docs/titan/1.0.0/getting-started.html)
为模板来演示HugeGraph的使用方法。通过对比HugeGraph和
[Titandb Getting Started](http://s3.thinkaurelius.com/docs/titan/1.0.0/getting-started.html)
可以了解HugeGraph和Titandb的差异。

## 1.1 HugeGraph与Titandb的异同

HugeGraph和Titandb都是基于[Apache TinkerPop 3](https://tinkerpop.apache.org)框架的图数据库，均支持[Gremlin](https://tinkerpop.apache
.org/gremlin.html)图查询语言，在使用方法和接口方面具有很多相似的地方。然而HugeGraph是全新设计开发的，其代码结构清晰，功能较为丰富，接口更为友好等特点。

HugeGraph相对于Titandb而言，其主要特点如下：

* HugeGraph拥有较为完善的工具组件。HugeGraph目前有HugeApi、HugeClient、HugeLoader、HugeStudio、HugeSpark等完善的工具组件，可以完成系统集成、数据载入、图可视化查询、Spark
连接等功能；

* HugeGraph具有Server和Client的概念，第三方系统可以通过jar引用、client、api等多种方式接入，而Titandb仅支持jar引用方式接入。

* HugeGraph的Schema需要显式定义，所有的插入和查询均需要通过严格的schema校验，目前暂不支持schema的隐式创建。

* HugeGraph充分利用后端存储系统的特点来实现数据高效存取，而Titandb以统一的Kv结构无视后端的差异性。

* HugeGraph的更新操作可以实现按需操作（例如：更新某个属性）性能更好。Titandb的更新是read and update方式。

* HugeGraph的VertexId和EdgeId均支持拼接，可实现自动去重，同时查询性能更好。Titandb的所有Id均是自动生成，查询需要经索引。


## 1.1 人物关系图谱

本示例通过Property Graph Model图数据模型来描述希腊神话中各人物角色的关系（也被成为人物关系图谱），具体关系详见下图。


 ![image](/images/graph-of-gods.png)


其中,圆形节点代表实体(Vertex)，箭头代表关系（Edge），方框的内容为属性。

该关系图谱中有两类顶点，分别是人物（character）和位置（location）如下表：

|名称|类型 |属性|
|---|---|---|
|character|vertex|name,age,type|
|location|vertex|name|


有六种关系，分别是父子（father）、母子（mother）、兄弟（brother）、战斗（battled）、居住(lives)、拥有宠物（pet）
关于关系图谱的具体信息如下：

|名称|类型 |source vertex label|target vertex label|属性|
|---|---  | ---|---  | ---|
|father|edge|character|character|-|
|mother|edge|character|character|-|
|brother|edge|character|character|-|
|pet|edge|character|character|-|
|lives|edge|character|location|reason|

HugeGraph需要进行严格的schema校验，对于edge label而言需要明确每一个edge label的source vertex label和target vertex label，
且相同的一组source vertex label和target vertex label只能有一个唯一的edge label。
因此，如果character和character有一种关系为father，在同一个图内部god和human之间的边就不能叫father。

关于`Edge Label` 强约束 "相同的一组`src vertex label` 和`tgt vertex label`只能有一个唯一的`edge label`" 的原因有2个：

 1. 如果`source`和`target`有多种`edge label`, 同时每一种`edge label`拥有不同的属性，那么`edge
 label`的属性就可能产生歧义，无法区分属性到底是属于哪条`edge`的。

 2. HugeGraph的`edgeId`是通过`src+label+tgt`拼接而成，同时HugeGraph支持多种Id生成策略，在解析`edgeId`的时候需要获取`vertex
label`中的Id策略配置才能正确解析`vertexId`。
在`edge label` 满足 " 相同的一组`source vertex label`和`target vertex label`只能有一个唯一的`edge label` "约束的前提下，
可以通过`edge label`获取唯一的`src vertex label`和`tgt vertex label`，而不需要冗余存储`src vertex label`和`tgt vertex label`信息。

因此本例子将原Titandb中的monster, god, human, demigod均使用相同的`vertex label: character`来表示, 同时增加属性type来标识人物的类型。
`edge label`与原Titandb保持一致。当然为了满足`edge label`约束，也可以通过调整`edge label`的`name`来实现。

## 2. Graph Schema and Data Ingest Examples
HugeGraph需要显示创建Schema，因此需要依次创建PropertyKey、VertexLabel、EdgeLabel，如果有需要索引还需要创建IndexLabel。

### 2.1 Graph Schema

```
schema = hugegraph.schema()

schema.propertyKey("name").asText().ifNotExist().create()
schema.propertyKey("age").asInt().ifNotExist().create()
schema.propertyKey("time").asInt().ifNotExist().create()
schema.propertyKey("reason").asText().ifNotExist().create()
schema.propertyKey("type").asText().ifNotExist().create()

schema.vertexLabel("character").properties("name", "age", "type").primaryKeys("name").nullableKeys("age").ifNotExist().create()
schema.vertexLabel("location").properties("name").primaryKeys("name").ifNotExist().create()

schema.edgeLabel("father").link("character", "character").ifNotExist().create()
schema.edgeLabel("mother").link("character", "character").ifNotExist().create()
schema.edgeLabel("battled").link("character", "character").properties("time").ifNotExist().create()
schema.edgeLabel("lives").link("character", "location").properties("reason").nullableKeys("reason").ifNotExist().create()
schema.edgeLabel("pet").link("character", "character").ifNotExist().create()
schema.edgeLabel("brother").link("character", "character").ifNotExist().create()
```

### 2.2 Graph Data

```
// add vertices
Vertex saturn = graph.addVertex(T.label, "character", "name", "saturn", "age", 10000, "type", "titan")
Vertex sky = graph.addVertex(T.label, "location", "name", "sky")
Vertex sea = graph.addVertex(T.label, "location", "name", "sea")
Vertex jupiter = graph.addVertex(T.label, "character", "name", "jupiter", "age", 5000, "type", "god")
Vertex neptune = graph.addVertex(T.label, "character", "name", "neptune", "age", 4500, "type", "god")
Vertex hercules = graph.addVertex(T.label, "character", "name", "hercules", "age", 30, "type", "demigod")
Vertex alcmene = graph.addVertex(T.label, "character", "name", "alcmene", "age", 45, "type", "human")
Vertex pluto = graph.addVertex(T.label, "character", "name", "pluto", "age", 4000, "type", "god")
Vertex nemean = graph.addVertex(T.label, "character", "name", "nemean", "type", "monster")
Vertex hydra = graph.addVertex(T.label, "character", "name", "hydra", "type", "monster")
Vertex cerberus = graph.addVertex(T.label, "character", "name", "cerberus", "type", "monster")
Vertex tartarus = graph.addVertex(T.label, "location", "name", "tartarus")

// add edges
jupiter.addEdge("father", saturn)
jupiter.addEdge("lives", sky, "reason", "loves fresh breezes")
jupiter.addEdge("brother", neptune)
jupiter.addEdge("brother", pluto)
neptune.addEdge("lives", sea, "reason", "loves waves")
neptune.addEdge("brother", jupiter)
neptune.addEdge("brother", pluto)
hercules.addEdge("father", jupiter)
hercules.addEdge("mother", alcmene)
hercules.addEdge("battled", nemean, "time", 1)
hercules.addEdge("battled", hydra, "time", 2)
hercules.addEdge("battled", cerberus, "time", 12)
pluto.addEdge("brother", jupiter)
pluto.addEdge("brother", neptune)
pluto.addEdge("lives", tartarus, "reason", "no fear of death")
pluto.addEdge("pet", cerberus)
cerberus.addEdge("lives", tartarus)
```

### 2.3 Indices

HugeGraph默认是自动生成Id，如果用户通过`primaryKeys`指定`VertexLabel`的`primaryKeys`字段列表后，`VertexLabel`的Id策略将会自动切换到`primaryKeys`策略。
启用`primaryKeys`策略后,HugeGraph通过`vertexLabel+primaryKeys`拼接生成`VertexId`
，可实现自动去重，同时无需额外创建索引即可以使用`primaryKeys`中的属性进行快速查询。
例如 "character" 和 "location" 都有`primaryKeys("name")`属性，因此在不额外创建索引的情况下可以通过`g.V().hasLabel('character')
.has('name','hercules')`查询vertex 。

> 注： 推荐使用[HugeStudio](http://hugegraph.baidu.com/quickstart/hugestudio.html)
> 通过可视化的方式来执行上述代码。另外也可以通过HugeClient、HugeApi、GremlinConsole和GremlinDriver等多种方式执行上述代码。

## 3. Graph Traversal Examples

### 3.1 Traversal Query

**1. Find the grand father of hercules**

```
g.V().hasLabel('character').has('name','hercules').out('father').out('father')
```

也可以通过`repeat`方式：

```
g.V().hasLabel('character').has('name','hercules').repeat(__.out('father')).times(2)
```

**2. Find the name of hercules's father**

```
g.V().hasLabel('character').has('name','hercules').out('father').value('name')
```

**3. Find the characters with age > 100**

```
g.V().hasLabel('character').has('age',gt(100))
```

**4. Find who are pluto's cohabitants**

```
g.V().hasLabel('character').has('name','pluto').out('lives').in('lives').values('name')
```

**5. Find pluto can't be his own cohabitant**

```
pluto = g.V().hasLabel('character').has('name', 'pluto')
g.V(pluto).out('lives').in('lives').where(is(neq(pluto)).values('name')

// use 'as' 
g.V().hasLabel('character').has('name', 'pluto').as('x').out('lives').in('lives').where(neq('x')).values('name')
```

**6. Pluto’s Brothers**

```

pluto = g.V().hasLabel('character').has('name', 'pluto').next()

// where do pluto's brothers live?
g.V(pluto).out('brother').out('lives').values('name')

// which brother lives in which place?
g.V(pluto).out('brother').as('god').out('lives').as('place').select('god','place')

// what is the name of the brother and the name of the place?
g.V(pluto).out('brother').as('god').out('lives').as('place').select('god','place').by('name')
```

推荐使用[HugeStudio](http://hugegraph.baidu.com/quickstart/hugestudio.html)
通过可视化的方式来执行上述代码。另外也可以通过HugeClient、HugeApi、GremlinConsole和GremlinDriver等多种方式执行上述代码。

### 3.2 总结

HugeGraph目前支持Gremlin的语法，用户可以通过Gremlin语句实现各种查询需求，但是目前HugeGraph的暂不支持'Or'类型查询和全文检索功能，因此也不支持多label查询。
和Titandb相比HugeGraph不支持的查询语句包括：

1. `g.V(hercules).out('father', 'mother')`

2. `g.E().has('reason', textContains('loves'))`