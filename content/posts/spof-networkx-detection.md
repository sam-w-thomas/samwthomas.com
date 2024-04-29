---
title: "SPOF NetworkX Detection"
date: 2024-04-15T20:46:47+01:00
slug: 2024-04-15-spof-networkx-detection
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

The ability to think of a TCP/IP Network as just a mathematical graph, is something that is underutilised by typical (automation) engineers in my opinion. Graph theory is a well established field of mathematical, and converting a network back to it's graphical state enables us to reap benefits at minimal cost.

In this blog post, I will introduce converting a (basic) network to a graph, at the IPv4 layer, using simply .cfg files. We can then manuplate this graph to detect Single Points of failure. 
This will be the first post in using graphs to gain visibility in your network. 

GitHub Repository for NetApollo,  Python implementation of the approach discussed: https://github.com/sam-w-thomas/NetApollo

## Graph Theory
Many Network Engineers, without a background in Computer Science, may scratch their head at what a 'graph' is? That's just a bar chart or something right? 
In the world of Computer Science, it has a different meaning - explained below from MIT.

<p style="text-align: center;">
"Informally, a graph is a bunch of dots and lines where the lines connect some pairs of dots"

"Graphs are ubiquitous in computer science because they provide a handy way to represent a relationship between pairs of objects. The objects represent items of interest such as programs, people, cities, or web pages, and we place an edge between a pair of nodes if they are related in a certain way"
- MIT ## 6.042J Chapter 5: Graph theory
</p>

Graphs come in two typical flavours:
* Directed Graphs - each edge has a concept of direction
* Undirected Graphs - each edge has no concept of direction
  
![enter image description here](/graph.PNG)

Here is the graph, but it's in it's (physical) network form.
![enter image description here](/network_graph.PNG)

For the purposes of simplicity, all graphs in this post will be Directed Graphs. Although, we could use a Directed Graph to represent receive and transmit. 

## Converting a TCP/IP network to a graph - Theory

Now, a network isn't something that you can typically convert to just a single graph; We have control and data planes for a reason. 

However, we can convert parts of a network to a single graph. For example, the IPv4 layer or the [OSPF Link State Database](https://hbristoweu.wordpress.com/2020/09/19/visualising-the-ospf-lsdb-with-python-textfsm-networkx-and-pygraphviz-and-yaml-dumping-the-lsdb/). We will create a single graph at the IPv4 layer.

However, to increase complexity and ability, you could layer graphs. As you'd expect, graph theory already has a word and setup for that already - Multilayer graphs. 
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
  
We now have a table of the IPv4 addressing for an entire network. Using scale techniques, such as [Pandas](https://pandas.pydata.org/) Dataframes and [Numpy](https://numpy.org/devdocs/index.html/), we could increase the efficiency of processing to handle large furthermore - due to the simplicty of the data structured, processing could be split into multiple nodes, further increasing peformace/scalability. 

### Fitering interfaces based on state


### Determining neighbors using IPv4 addressing

Iterating through the interface-address table produced, enables us to build an adjacency list. An adjacency list of the network described above looks like the following:  
R1: R2, R3    
R2: R1, R3  
R3: R1, R4, R5  
R4: R3, R5  
R5: R3, R4  

Please note, we are looking at this at the IPv4 layer. Whilst R4, R5 and R3 share an NBMA network segment, we simplify for our purposes. 

This adjacency list can then be converted to a *NetworkX graph*.
  
To determine the adjacency list from the interface-address table produced, we follow the following logic:
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

Here is when we are introduced to NetworkX. 
NetworkX describes itself as ".. a Python package for the creation, manipulation, and study of the structure, dynamics, and functions of complex networks". 
It provides the basis for many graph manipulation, without us having to do the hardwork. 

To load our adjanecy list as a NetworkX Graph, we only need the following code:
```python
import networkx as nx

networkx_graph = nx.from_dict_of_lists(adj_list)
```

### Iterate through graph to find SPoFs
