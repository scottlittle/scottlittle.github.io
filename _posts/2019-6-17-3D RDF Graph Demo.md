---
layout: post
title: 3D RDF Graph Demo
---

While learning `SPARQL` ([see previous post](http://scottlittle.org/Learning-SPARQL/)) I explored the tutorials of the [Brick Schema](http://brickschema.org) to get a handle on making queries on an RDF database (using the `RDFlib` python package).  One small complaint about `RDFlib` is that there is not an obvious way of outputting query results as a nice table.  I learned, however, that the output can be outputted to a JSON string and from there easily turned into a dictionary or pandas dataframe.  Additionally, you can programmatically replace the prefixes:

s|p|o
---|---|---
rice: Exhaust_Air_Tempdac4UIP22|rdf: type|brick: Exhaust_Air_Temperature_Sensor
rice: Room101|bf: hasPoint|rice: TEMP2_Space_Temperature_RMI101
rice: FP_VAV_Hot_Water_Return_Temp|rdf: type|brick: Hot_Water_Return_Temperature
rice: RMI536_Space_Temperature_Perimeter|rdf: type|brick: Room_Temperature_Sensor
brick: Freeze_Protect_Low_Limit|rdf: type|brick: Point

This sort of output is easier to read than row by row output that the tutorials use.  From this output, I can construct a graph (using `igraph`) and view the results.  I built a custom 3D graph framework using `pythreejs` so that I could explore the graph in 3D:

![](https://raw.githubusercontent.com/scottlittle/scottlittle.github.io/master/images/brick-movie-480.gif)

[You can explore this for yourself here.](http://scottlittle.org/brick-app-export)  This version is static in the sense that it is pre-rendered to decrease loading time.  A dynamic version that you could query, using SPARQL for instance, or select nodes with the mouse is possible if the application required it.
