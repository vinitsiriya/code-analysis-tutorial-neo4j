# C# Code Analysis using Neo4j!

Let's start by using  **[csharp-call-graph](https://github.com/vinitsiriya/csharp-call-graph)** tool . If you want to learn about find dependencies among code, you can use this tool. If you want to play with Dependency Graph, Call Graph, you can use this tool to do lot of code analysis with C#. Once you have finished this tutorial me, you can do your code analysis like pro.


# Tutorial

Well after getting your database ready with the tool  **[csharp-call-graph](https://github.com/vinitsiriya/csharp-call-graph)**. You are ready with the call graph. 
> *Note: Call Graph is graph of call the method invocations only. This tutorial is about transforming and using that information for further analysis.*

## Understanding our Call Graph

You can see call graph in neo4j after using our tool. We have

## Node

We have node  which have label named **:Method**. It have many properties like *name, class_name, assembly, containing file path* etc. Please have a look at details about the properties in  **[csharp-call-graph](https://github.com/vinitsiriya/csharp-call-graph)**

## Relationship
We have **METHOD_CALL** named relationship which have no attribute.  It represents a method invocation.


## First Query

You can query the database of neo4j created by our tool using.
```
(m1:METHOD)-[r:METHOD_CALL]->(m2:METHOD) RETURN m1,r,m2 LIMIT 25
```
>  You can also use Bloom to see more detailed view of this. You have to remove LIMIT if you are using Bloom.


# Class Dependency Diagram using Call Graph 

Class Dependency is one of the biggest features for our code analysis. It enables you to get a view how a particular class is related with other classes stored in your project. This allows you to keep analyzing easily into your project directories... The class dependency diagram based on call graph can enable you to get the starting point to read source code as we can see hierarchies easily

There are two types of dependency analysis for classes.

- Dependency based on Inheritance.  
	> We are not covering this yet with our tool. As we find dependency based on call graph more useful

- Dependency based on Call Graph 
	> The most important part of this analysis is that we can know which class is consumer of which class and which class is giving service to which class. This may help us to know about the conversation happening between them

Here are the steps for creating  class dependency  using call graph.
## Creating  Class Dependency Nodes in Neo4j

####  Step 1 : Copy class name to temporary property  
```
 MATCH (n:Method) set n.class_name_temp = n.class_name
```

####  Step 2 : Copy class name to temporary property  
```
CREATE CONSTRAINT ON (n:`Class`) ASSERT n.`name` IS UNIQUE
```


####  Step 3 : Copy class name to temporary property
Before running this query make sure  that neo4j.conf has these two lines to avoid unnecessary errors  

>dbms.security.procedures.unrestricted=apoc.*
dbms.security.procedures.whitelist=apoc.*  
```
CALL apoc.refactor.categorize('class_name_temp', 
'MEMBER_OF', true, 'Class', 'name',
 ['solution_id','project_name','project_id'], 1)

```

####  Step 4 : Create a  METHOD_CALLS_TO relation between classes

METHOD_CALLS_TO relation going to store information about number of class of different type of methods to different type in another class.
 
We are going to do this in step wise manner. 

```
match (n:Class)<--(m:Method)-->(m2:Method)-->(n2:Class)
create (n)-[r:USES{method_call:1}]->(n2)
set r.ss_call = 
		CASE
    	WHEN (m.method_symbol_isStatic ="True" and m2.method_symbol_isStatic="True") 
        THEN 1
    	ELSE 0
    	END
        
set r.ms_call = 
		CASE
    	WHEN (m.method_symbol_isStatic ="False" and m2.method_symbol_isStatic="True") 
        THEN 1
    	ELSE 0
    	END
set r.sm_call=
	    CASE
    	WHEN (m.method_symbol_isStatic ="True" and m2.method_symbol_isStatic="False")  
        THEN 1
    	ELSE 0
    	END
set r.mm_call=
		CASE
    	WHEN (m.method_symbol_isStatic ="False" and m2.method_symbol_isStatic="False") 
        THEN 1
    	ELSE 0
    	END

set r.abstract_call= 
		CASE
    	WHEN (m.method_symbol_isAbstract="True") 
        THEN 1
    	ELSE 0
    	END
 set r.virtual_call=
 		CASE
    	WHEN (m.method_symbol_isVirtual="True") 
        THEN 1
    	ELSE 0
    	END

```
```
MATCH (c:Class)-[r:USES]->(c) 
delete r 
```

```
Match (c1:Class)
Match(c2:Class)
Match (c1)-[r:USES]->(c2)
with c1,c2,sum(r.method_call) as method_call_count, sum(r.mm_call) as mm_call_count, sum(r.ms_call) as ms_call_count, sum(r.sm_call) as sm_call_count, sum(r.ss_call) as ss_call_count MERGE (c1)-[r:METHOD_CALLS_TO]->(c2)
```

```
match (:Class)-[r:USES]->(:Class) 
delete r
```

Your data with class dependency diagram is ready for analysis. Enjoy !


## Class Hierarchy Analysis using Class Dependency Nodes in Neo4j

After creating class dependency diagram in Neo4j you are ready for obtaining more knowledge about the code structure. Let's get started with subgraph of class dependency diagram.

## Subgraph in Class Depedency 

A subgraph here represent how a particular class is dependent on classes it use. Lets quickly see how to get it done in Neo4j.

```
MATCH (p:Class) where p.name contains "AnalyzerDriver"

CALL apoc.path.subgraphAll(p, {	relationshipFilter: "METHOD_CALLS_TO",    minLevel: 0,    maxLevel: 1})
YIELD nodes, relationships

UNWIND relationships as rel
RETURN startNode(rel),rel,endNode(rel)
```
> **Extra Notes**
>   *  We are using **UNWIND** so that it can support Bloom visualization also.

I will be adding more queries soon. Till then, Enjoy the power of code analysis. 

