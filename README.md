#Allengheny County Crashes

This project attempts to visualize and analyze Allegheny County crashes from 2010 - 2014. The crash data provided by the county includes crash points. But if we want to know about how "dangerous" a road is, we also need to know the level of usage of the roads on which crashes occur. Roads with more usage cars are more likely to have crashes simply because there is a larger volume of traffic.

Luckily, PennDOT provides the PA State Roads data, which includes road segment shapefiles and Average Average Daily Traffic Counts. By merging this data with crashes that are relatively close to each road, we can get a measure of how dangerous a road is. Then we can map then and start to ask questions about why more crashes occur there. From there further analyses can be conduct at the road segment level to investigate the effects of road geometry and crash conditions on the expected number of crashes using statistical methods like multiple regression. 

To get started with visualization, I've created a couple of quick maps to see what the data look like:

#2014 Crashes by Severity and Number of Injuries
[https://scotter.github.io/crashes/2014_crashes.html](https://scotter.github.io/crashes/2014_crashes.html)
This map shows 2014 crashes with injuries. The size of each circle is related to the number of injuries and the color is related to the max severity level.

#2014 Crash Animation by Time of Day
[https://scotter.github.io/crashes/animation.html](https://scotter.github.io/crashes/animation.html)
This map animates 2014 crashes with injuries by time of day. Again, the size of each circle is related to the number of injuries and the color is related to the max severity level.

#Average Annual Daily Traffice
[https://scotter.github.io/crashes/aadt.html](https://scotter.github.io/crashes/aadt.html)
This map shows the Average Annual Daily Traffic for road segments present in the PA State Roads data. Note that this only includes state roads. Unfortunately, we lack data on traffic volumes for local roads.


