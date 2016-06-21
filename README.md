#Allengheny County Crashes

This project attempts to visualize and analyze Allegheny County crashes from 2010 - 2014. The crash data provided by the county includes crash points. But if we want to know about how "dangerous" a road is, we also need to know the level of usage of the roads on which crashes occur. Roads with higher levels of usage are more likely to have crashes simply because there is more traffic and more opportunity for a crash.

Luckily, PennDOT provides the PA State Roads data, which includes road segment shapefiles and Annual Average Daily Traffic Counts. By merging this data with crashes that are close to each road, we can get a measure of how dangerous a road is. Then we can map them and start to ask questions about why more crashes occur there. Further analyses can be conducted to investigate the effects of road geometry and crash conditions on the expected number of crashes using statistical methods like multiple regression. 

To get started with visualization, I've created a couple of quick maps to see how the data look:

#2014 Crashes by Severity and Number of Injuries
<a href="https://scotter.github.io/crashes/2014_crashes.html" target="blank">https://scotter.github.io/crashes/2014_crashes.html</a>

This map shows 2014 crashes with injuries. The size of each circle is proportional to the number of injuries and the color is related to the max severity level.

#2014 Crash Animation by Time of Day
<a href="https://scotter.github.io/crashes/animation.html" target="blank">https://scotter.github.io/crashes/animation.html</a>

This map animates 2014 crashes with injuries by time of day. Again, the size of each circle is related to the number of injuries and the color is related to the max severity level.

#Annual Average Daily Traffic
<a href="https://scotter.github.io/crashes/aadt.html" target="blank">https://scotter.github.io/crashes/aadt.html</a>

This map shows the Annual Average Daily Traffic for road segments present in the PA State Roads data. Note that this only includes state roads. Unfortunately, we lack data on traffic volumes for local roads.

#Vehicle Miles Traveled
<a href="https://scotter.github.io/crashes/aadt.html" target="blank">https://scotter.github.io/crashes/vmt.html</a>

This map shows the Vehicle Miles Traveled for road segments present in the PA State Roads data. VMT adjusts for the length the of the road segment. As above, note that this only includes state roads.

#Road Segments By Crash Rates
<a href="https://scotter.github.io/crashes/aadt.html" target="blank">https://scotter.github.io/crashes/segments_and_crashes.html</a>

This map shows the road segments colored by their Fatal and Major Injury <a href="http://safety.fhwa.dot.gov/local_rural/training/fhwasa1109/app_c.cfm" target="_blank">rate per Vehicle Miles Traveled</a>. Hovering on segment shows the associated crashes in from 2010 - 2014. This is the visualization I'm working toward and it's still very much a work in progress. Once I finish up computing the stats, I'd like to walk the user through the most dangeous or risky state roads, showing the road name, the number of crashes, and maybe even a Google Streetview view of the road where the crashes occured.

The process for getting the data is in the [data_wrangling.md](https://github.com/scotter/crashes/blob/master/data_wrangling.md) file. Originally, I started with crashes, aggregated them to road segments, then aggregated those. In retrospect, I should have started with the road geometry data, averaged the traffic volumes over 5 years, selected the most recent segment geomtry, and then assigned the crashes to a segment and computed stats. I hope to do this before continuing with the visualization.