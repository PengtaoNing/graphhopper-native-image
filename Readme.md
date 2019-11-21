# Create Native Image for GraphHopper

 * [download graalvm](https://github.com/graalvm/graalvm-ce-builds/releases/)
   for your OS e.g graal CE for linux amd64, 19.3.0 and extract to `graalvm/`
 * run `bin/gu install native-image`
 * The native-image tool needs a local C toolchain: glibc-devel, zlib-devel and gcc - do e.g.`sudo apt-get install zlib1g-dev`
 * verify that the tool works: `$JAVA_HOME/bin/native-image --help`
 * set JAVA_HOME to the downloaded graal (`graalvm/`) to use the correct JVM for maven
 * `mvn clean package` => should create a native image at ./target/graphhopper

# Start the GraphHopper Native Image

Now that graal created your native image you can start graphhopper:

 * download OSM road data: `wget http://download.geofabrik.de/europe/germany/berlin-latest.osm.pbf -O osm.pbf`
 * start GraphHopper: `./target/graphhopper`

It will take a few seconds where it basically converts the pbf
file into an internal graph structure (see graph-cache folder), but all subsequent
calls will be very fast and calculate the distance between two points in
Berlin (see java code).

If you are finished and you noticed something is eating your CPU resources do:

`$JAVA_HOME/bin/native-image --server-shutdown`

# Unscientific Results

Measured via command line utility `time` and picked the "real" time

 * OpenJDK: 0.5s
 * native image: <0.05s
 * shared library: ~0.025s
 
# Future Tasks

 * try --shared or and --static options of `native-image` and include in a cpp application!
 * target iOS
 * target Android

# How to make resources of dependencies working?

Run application with GraalVM and special agent:

```
$JAVA_HOME/bin/java -agentlib:native-image-agent=config-output-dir=./ni-configs -jar target/*-jar-with-dependencies.jar
```

Then include these files in native-image command line: see maven plugin.

See also:

https://github.com/oracle/graal/blob/master/substratevm/CONFIGURE.md

https://github.com/oracle/graal/blob/master/substratevm/RESOURCES.md

# Open Questions

 * slf4j-log4j bridge fails with ClassNotFound. Workaround: use slf4j-simple. Likely this is fixed via the resources config in previous section or maybe
   [this](https://github.com/oracle/graal/issues/653)
 * Why are they e.g. libgraphhopper_dynamic.h headers and how can I use 
   `--static` as then no headers are created?

# Call GraphHopper from C++

Instead of a native application we now want a library that is called from
native code like example.cpp. To create this library do the following:

 * use `<imageName>libgraphhopper</imageName>` and add `--shared` in `<buildArgs>` in pom.xml
 * `mvn clean package`

Now you should see libgraphhopper.so, libgraphhopper.h, graal_isolate.h in the
./target folder. Proceed with:

```
g++ example.cpp -I ./target -L ./target -lgraphhopper -o target/example
export LD_LIBRARY_PATH=./target:$LD_LIBRARY_PATH && ./target/example 52.5147 13.3
[main] INFO com.graphhopper.reader.osm.GraphHopperOSM - version 1.0|2019-11-20T22:11:41Z (5,15,4,3,5,5)
[main] INFO com.graphhopper.reader.osm.GraphHopperOSM - graph CH|car|MMAP_STORE|2D|no_turn_cost|5,15,4,3,5, details:edges:126,973(4MB), ...
Distance 6806.26
```

# Graal Help
 
 * installing the native-image plugin fails? Download the jar, then do `bin/gu -L install ~/Downloads/native-image-installable-svm-java11-linux-amd64-19.3.0.jar`
 * https://www.graalvm.org/community/
 * https://github.com/graalvm/graalvm-demos
