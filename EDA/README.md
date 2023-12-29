# EDA Observations

Thought: The dataset contains lines from outside the contiguous US, so I will work with a filtered version by plotting the GeoJSON and determing what coordinates to bound our dataset by.

Result: filter by [-130:-40, 25:55]. Even after filtering, there are still a large number of edges and vertices - about 90,000 and 74,000, respectively. This will be important as we consider how to find crucial sections of the grid, as a metric like eigencentrality will take a long time to run.

---
Thought: To convert the transmission lines dataset into a vertex-edge graph, we need to define nodes. The dataset contains only lines, so nodes would naturally be the points where transmission lines intersect and edges would be the lines themselves. Solution: extract the first and last point of each transmission line and construct a graph using these as vertices and the lines as edges. We can do this with graph-tool and easily make coordinates, line lengths, and line voltages properties of either vertices or edges. Ideally, this will result in a completely connected graph.

Result: There are 775 components. The largest has 97% of the nodes. This is probably enough enough to begin the rest of the project, but I want to know more about my data quality first. I will investigate the other components to see why the graph isn't connected.

---
Observation: Some states have uploaded their own transmission line data, and that data isn't consisent with the US Energy Atlas even though the Atlas takes some of its information from them. To have the most accurate, up-to-date dataset, I would need to incorporate additional data sources.

---
Observation: Some edges that I know share a connection (from the ArcGIS map) don't have a shared start/end coordinate in the GeoJSON. This means that I'm missing some existing edges. On the plus side, there are two fields: SUB_1 and SUB_2 which denote substations that the transmission line connects to. Using these fields, as well as the start/end coords of transmission lines, I can make a more complete list of vertices to construct edges between.

Update: A number of complications have arisen since I started looking into this problem:
- Lines that connect at a common substation don't always share a coordinate pair.
- Lines that connect at a common substation don't always share a substation name.
- A substation can have two different coordinates associated with it because it is large enough for transmission lines to connect on either end.
- Lines that connect in real life might not share coordinates or a substation name in the data.
I'm not sure how to tackle this yet, so I might test some algorithms for finding weak points on the grid before committing to a solution.