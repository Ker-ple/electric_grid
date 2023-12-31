# EDA Observations

#### Thought: 
The dataset contains lines from outside the contiguous US, so I will work with a filtered version by plotting the GeoJSON and determing what coordinates to bound our dataset by.

#### Result: 
Filter by [-130:-40, 25:55]. Even after filtering, there are still a large number of edges and vertices - about 90,000 and 74,000, respectively. This will be important as we consider how to find crucial sections of the grid, as a metric like eigencentrality will take a long time to run.

---
#### Thought: 
To convert the transmission lines dataset into a vertex-edge graph, we need to define nodes. The dataset contains only lines, so nodes would naturally be the points where transmission lines intersect and edges would be the lines themselves. Solution: extract the first and last point of each transmission line and construct a graph using these as vertices and the lines as edges. We can do this with graph-tool and easily make coordinates, line lengths, and line voltages properties of either vertices or edges. Ideally, this will result in a completely connected graph.

#### Result: 
There are 775 components. The largest has 97% of the nodes. This is probably enough enough to begin the rest of the project, but I want to know more about my data quality first. I will investigate the other components to see why the graph isn't connected.

---
#### Observation: 
Some states have uploaded their own transmission line data, and that data isn't consisent with the US Energy Atlas even though the Atlas takes some of its information from them. To have the most accurate, up-to-date dataset, I would need to incorporate additional data sources.

---
#### Observation: 
Some edges that I know share a connection (from the ArcGIS map) don't have a shared start/end coordinate in the GeoJSON. This means that I'm missing some existing edges. On the plus side, there are two fields: SUB_1 and SUB_2 which denote substations that the transmission line connects to. Using these fields, as well as the start/end coords of transmission lines, I can make a more complete list of vertices to construct edges between.

#### Update: 
A number of complications have arisen since I started looking into this problem:
- Lines that connect at a common substation don't always share a coordinate pair.
- Lines that connect at a common substation don't always share a substation name.
- A substation can have two different coordinates associated with it because it is large enough for transmission lines to connect on either end.
- Lines that connect in real life might not share coordinates or a substation name in the data.
I'm not sure how to tackle this yet, so I might test some algorithms for finding weak points on the grid before committing to a solution.

#### Thought: 
Ideally, I could calculate pairwise distances between all of my coordinates and group coordinates that are within $\epsilon$-tolerance of each other. These would denote substations. Unfortunately, I have about 74k unique coordinates to work with, so this is computationally infeasable. Storing the points as doubles would also take about 45 gigabytes of space.

---
#### Update: 
I have devised an imperfect method for quickly clustering nearby coordinates together into the correct substations. 

#### Context: 
We need to identify vertices for our vertex-edge graph representation of the transmissions grid.
Problem: To distinguish vertices that are close together, we need to use a combination of substation names and coordinates. As multiple substations can share a name, multiple substations can be near each other with different names, a single substation can fail to have common coordinates because of data quality issues, coordinates can connect but incorrectly not share a substation name, and we only have access to edge-level data (which contains name and coord information about their endpoints but the connection isn't indicated at row-level data), this is quite complex.

Examples of problem nodes (see images/):
- PINEVILLE, which is a large substation and has transmission lines connecting to it at two different coordinates. This substation should be classified as one vertex on our graph, and each edge that connects to either of the two points should connect to the one vertex.
- MIDWAY, which is a name that belongs to 3(?) substations across the US. 
- A location in Louisiana's southern islands has three points which should connect but don't their coordinates don't quite meet. 

#### Solution: 
Round each substation's coordinates to 2 decimal points and COUNT the lines that have it as an endpoint. This will turn cases like PINEVILLE into a single substation. From this level of precision, there are still some coordinates that are very near to each other yet not grouped together, so there's still the question of which one corresponds to the substation. These could be the other ends of the transmission lines or other parts of the substation. So, we need to aggregate each substation's coordinates to 1 decimal point and COUNT because that is coarse enough to group all of these possibilities together into one point. Join these two views together on substation and coordinates rounded to 1 decimal point and calculate the two COUNTs from before. The proper substation coordinate is the coordinate in the row with the highest COUNT for each substation-coarse-coordinate combo. 

#### Result: 
This isn't quite right because some substations that have two coordinates have many connections to both coordinates, so though each connection is a plurality they are not majorities (see CHESTERFIELD screenshot). I might need to have a more clever aggregation system than simply rounding. This is furthered by the fact that some stations that are very close together simply aren't being rounded together as I'd like them to because ROUND is too simple an algorithm. For example, rounding to the tens place would put 34.9 and 35.1 in different bins, rounding to the nearest 5 would put 37.51 and 37.48 in different bins, and so on. I'm thinking of using a dual-bin system, where I make bins of coord-subs rounded to the nearest 2 and the nearest 1, and make a 2-part decision. If coord-sub combos are placed in different ones bins yet the same twos bin, then they go together, and if they go in the same ones bin they go together. Still this seems to be sus in cases on the edge, e.g. 37.1 and 38.9 would be placed in the 38s bin, but maybe they're too far apart? Perhaps I just need smaller bins, but this might be a problem inherent to the rounding method because you can always find problematic points on the boundaries.