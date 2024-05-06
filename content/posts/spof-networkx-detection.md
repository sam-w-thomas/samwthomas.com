---
title: "SPOF NetworkX Detection"
date: 2024-05-06T14:00:00+01:00
slug: spof-networkx-detection
type: posts
draft: false
categories:
  - automation
tags:
  - automation
  - graph
  - ipv4
  - networkx
  - ciscoconfparse
  - python
  - spof
---
<base href="{{ .Site.BaseURL }}">

# Detecting Single Points of Failure using NetworkX

The ability to think of a TCP/IP Network as just a mathematical graph is underutilised by typical (network automation) engineers in my opinion. Graph theory is a well established field of mathematics, and converting a (computer) network to it's graphical state enables us to reap benefits at minimal cost.

In this blog post, I introduce converting a (basic) network to a graph, at the IPv4 layer, using Cisco .cfg files. We can manuplate this graph to detect Single Points of Failure (SPoF) and audit network reliability.
This will be the first post in using graphs to gain visibility into (computer) networks.

GitHub Repository for NetApollo, Python implementation of the approach discussed: https://github.com/sam-w-thomas/NetApollo

Is is worthwhile pointing out, networking modelling is already a well-practiced disicpline. This is just my contribution! 

## Graph Theory
Many Network Engineers, without a background in Computer Science, may scratch their head at what a 'graph' is? That's just a bar chart or something right? 
In the world of Computer Science, it has a different meaning - explained below from MIT.

<p style="text-align: center;">
"Informally, a graph is a bunch of dots and lines where the lines connect some pairs of dots"

"Graphs are ubiquitous in computer science because they provide a handy way to represent a relationship between pairs of objects. The objects represent items of interest such as programs, people, cities, or web pages, and we place an edge between a pair of nodes if they are related in a certain way"
- [MIT ## 6.042J Chapter 5: Graph theory](https://ocw.mit.edu/courses/6-042j-mathematics-for-computer-science-fall-2010/f471f7b7034fabe8bbba5507df7d307f_MIT6_042JF10_chap05.pdf)
</p>

Graphs come in two _typical_ flavours:
* Directed Graphs - edges have a concept of direction
* Undirected Graphs - edges have no concept of direction
  
Example of a basic graph.
![Basic graph](/graph.PNG)

Here is the graph, but it's in it's (physical) network form.
![Graph covnereted to a (basic) physical view of a network](/network_graph.PNG)

For the purposes of simplicity, all graphs in this post will be undirected graphs. Although, we could use a directed Graph to represent receive and transmit. 

## Converting a TCP/IP network to a graph

Now, you can't convert a TCP/IP network into just a single graph; We have control, data and management planes for a reason. 

However, we can convert parts of a network to a single graph. For example, the IPv4 layer or the [OSPF Link State Database](https://hbristoweu.wordpress.com/2020/09/19/visualising-the-ospf-lsdb-with-python-textfsm-networkx-and-pygraphviz-and-yaml-dumping-the-lsdb/). We shall create a single graph at the IPv4 layer in this blog post.

However, to increase complexity and ability, you could layer graphs. As you'd expect, graph theory has a word and ability for this already - Multilayer graphs. 
![Example of a Multilayer Graph](https://i0.wp.com/braph.org/wp-content/uploads/2022/04/multiplex.png)

## Converting a TCP/IP network to a graph - Implementation

As mentioned above, we shall create a graph at the IPv4 layer, and use this to detect SPoFs. 
I break this problem down into the following domains:
1. Gathering IPv4 addressing
2. Filtering interfaces based on state (Optional)
3. Determining neighbors using IPv4 addressing and convert to adjacency list
4. Convert adjacency list into a graph
5. Iterate through graph to find SPoFs
  

### Gathering IPv4 addressing
Fundementally, there are two approaches we can use to gather addressing, neither are mutally exclusive:
* Configuration files
* State information (e.g. "show" commands in Cisco)

In [NetApollo](https://github.com/sam-w-thomas/NetApollo) I demonstrate using just configuration files. This has the following advantages:
* Enables a third-party to audit configuration with no live connectivity requirements (or equivilient)
* Provides a building block for other information. For example, we could *add* state information onto the configuration files

In NetApollo, IPv4 addressing is achieved using [ciscoconfparse](https://pypi.org/project/ciscoconfparse/). This scrapes the configuration files in a folder, and is used to produce the following structure for each interface:
![enter image description here](/ipv4_row.PNG)
  
If we apply this logic to an actual network, the one below, we get the following result(s):
![enter image description here](/ipv4_graph_example_2.PNG)  
![enter image description here](/ipv4_graph_example.PNG)
  
We now have a table of the IPv4 addressing for an entire network, an "interface-address table". Using scale techniques, such as [Pandas Dataframes](https://pandas.pydata.org/) and [Numpy](https://numpy.org/devdocs/index.html/), we could increase the efficiency of processing.
  

### Fitering interfaces based on state

I want to keep this relatively short, as I did not implement this approach.
However, you could use keys in the static configuration (e.g. interface names) and combine this with tools, such as paramiko/TextFSM, which would enable you to filter links based on a state of your choosing, such as `down`. Alternatively, you could also layer on information such as errors; increasing graph complexity to achieve aims. 
  

### Determining neighbors using IPv4 addressing

Iterating through the interface-address table produced, enables us to build an adjacency list. An adjacency list of the network above looks like:  
R1: R2, R3    
R2: R1, R3  
R3: R1, R4, R5  
R4: R3, R5  
R5: R3, R4  

Please note, we are looking at this at the IPv4 layer. R4, R5 and R3 share an NBMA network segment so there is more actual complexity; however, we simplify for our purposes. 

This adjacency list can then be converted to a (NetworkX graph)[https://networkx.org/].
  
To determine the adjacency list from the interface-address table produced, we follow the logic:
```goat
                                    +------------------------------------------------------------+                                                   
                                    |                                                            |                                                   
                                    |   Select Interface from interface-address table (Primary)  |  <---------------+                                
                                    |                                                            |                  |                                
                                    +-------------+----------------------------------------------+                  |                                
                                                  |                                                                 |                                
                                                  |                                                                 |                                
                                                  |                                                                 |                                
                                                  v                                                                 |                                
                                    +---------------------------------------------------------------+               |                                
                                    |                                                               |               |                                
                       +----------> |  Select Interface from interface-address table  (Comparison)  |               |                                
                       |            |                                                               |               |  Continue to next              
                       |            +-------------+-------------------------------------------------+               |  interface in interface-address
                       |                          |                                                                 |  table for outer-loop          
                       |                          v                                                                 |                                
                       |            +-------------------------------------------------+                             |                                
Continue to next       |            |                                                 |                             |                                
interface in           |            |  Perform XOR on Primary and Comparison using    |                             |                                
interface-address      |            |  address and subnmetmask to determine if segment|                             |                                
table for inner-loop   |            |  neighbors                                      |                             |                                
                       |            +-------------+-----------------------------------+                             |                                
                       |                          |                                   |                             |                                
                       |                          | Are Neighbors                     | Are Not Neighbors           |                                
                       |                          |                                   |                             |                                
                       |                          v                                   v                             |                                
                       |            +----------------------------+         +---------------------------+            |                                
                       |            |                            |         |                           |            |                                
                       |            |   Add Primary and          |         |  Last interface in        |            |                                
                       |            |   Comparison as neighbors  +-------> |  interface-address table  +------------+                                
                       |            |   on adjacency list        |         |                           |    YES                                      
                       |            |                            |         |                           |                                             
                       |            +----------------------------+         +------------+--------------+                                             
                       |                                                                |                                                            
                       |                                                                |                                                            
                       |                                                                |                                                            
                       |                            NO                                  |                                                            
                       +----------------------------------------------------------------+                                                            
```
In practice, the adjacency list takes the form of a dictonary of lists. This demonstrated below for the example topology shown above. 
```python
{
  "R1": ["R2", "R3"],
  "R2": ["R1", "R3"],  
  "R3": ["R1", "R4", "R5"],  
  "R4": ["R3", "R5"],  
  "R5": ["R3", "R4"]  
}
```
### Convert adjacency list into a graph

Here is when we are introduced to [NetworkX](https://networkx.org/). 
NetworkX describes itself as ".. a Python package for the creation, manipulation, and study of the structure, dynamics, and functions of complex networks". 
It provides the basis for (our) python graph manipulation, without us having to do the hardwork. 

To load our adjanecy list as a NetworkX Graph, we only need the following code:
```python
import networkx as nx

networkx_graph = nx.from_dict_of_lists(adj_list)
```
  

### Iterate through the graph to find Single Points of Failure

To identify SPoFs, we iterate through every node. Removing the node, and seeing if it is a disconnected graph.
A "disconencted graph" is a graph theory term which states the graph that has two distinct/seperate components, i.e. there are atleast two nodes with no path between them. 

Please note, when impelemnting this in python you need to use the `.copy()` function to create a seprate graph each iteration. Otherwise, you would eventially remove all nodes from your original graph; We want a fresh (original) graph every iteration.  
```python
spof_nodes = []
    for node in original_graph.nodes:
        iteration_graph = original_graph.copy()
        iteration_graph.remove_node(node)
        if not nx.is_connected(demo_graph):
            spof_nodes.append(node)
```

Using the above example, `spof_nodes` now represents all nodes which are SPoFs at the IPv4 layer.
  

## Scalability 

I don't want to delve to much into this, as I don't have the time to demonstrate this approach on 10,000's of nodes. However, given the simple data structures explored. I beleive this approach could be scaled up to networks with 100,000's of devices. This could *potentially* be achieved by breaking tasks and using tools such as [Apache Hadoop](https://hadoop.apache.org/). 
<br><br><br>

## Practical Applications

Whilst this approach could be used to pull static configuration files and quickly identify Single Points of Failure. There are more complete solutions which may be more appropiate for your particular needs. For example, [Batfish](https://batfish.org/) has a [comprehensive approach](https://batfish.readthedocs.io/en/latest/notebooks/linked/analyzing-the-impact-of-failures-and-letting-loose-a-chaos-monkey.html) to analyse the impact of network failure in greater complexity than is covered here. However, we hope this blog post can provide a reference for simpler analysis and fundemental learning. Furthermore, propiertry tools/products exist, such as [IPFabric](https://ipfabric.io/).
