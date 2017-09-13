---
title: "Unit testing a distributed service"
---
>Testing a distributed application in a realistic environment with multiple hosts over a network is a hassle, and not easy to do in a development environment with requirements for simple, robust, reproducible tests with a really quick turnaround. This post outlines a light-weight test setup, based on Docker and Docker Compose, that enables automated, self-contained and stateless testing of distributed software, that can be easily deployed and maintained on a developer laptop. It is demonstrated how to emulate scenarios with multiple network interfaces, host crash/restart, and network partition. 

[Example project.](https://github.com/mxns/hazelcast-lab)

## Unit testing a networked application?

Let us say you have built a distributed service and ascertained that everything works when all the nodes that make up your service are running on the same host, i.e. your developer machine. Now you would like to test how it behaves in a more realistic scenario, where the nodes are running on different host machines, connecting to each other over a network. Is consistency maintained after a host restart? Will your fail over strategy kick in when a host goes down? What really happens if there is a network partition? Scenarios that cannot be tried and tested by running the system on a single laptop - you need to set things up on multiple hosts on a real or emulated network.

We are looking for a method that lets us effortlessly try things out, using something that resembles a suite of unit tests: completely automated, with a quick turnaround, few outside dependencies, and with a clear result: did it fail or succeed? The time between changing a line of code or configuration, and seeing the change deployed in a realistic scenario with multiple host machines across a network, should be maximum a question of minutes, if possible seconds. The quicker and more effortless the procedure of launching and evaluating a test suite, the greater the coding pleasure, and the easier it will be to experiment with code changes and try out obscure configuration parameters: in other words, the easier to really get to know the ins and outs of your distributed system and how it behaves when the wind is strong and the sea is rough.

A test setup such as this can of course be achieved by setting up and provisioning real physical or virtual host machines, using manual labor or your orchestration tool of choice. The process can be highly automated using, say, Ansible, AWS, or VMWare. In a mature project the necessary processes will probably already be available to the testing team, perhaps integrated in a Continuous Delivery pipeline. Unlike these powerful techniques, the process described here is aimed at the developer looking for something that is easy to setup, modify and maintain, is self-contained and stateless, doesn't interfere with shared resources, and runs offline, just like a regular unit test. Simplicity and robustness is preferred over advanced features.

Let's see if we can achieve this using Docker and Docker Compose.

## General idea

Let us make the pretty general assumption that the project has a build process that produces artifacts from source code. A node is launched by deploying an artifact, produced by the build process, together with some node specific configuration. The configuration would typically specify, among other things, which network interfaces to use for intra-node communication. 

Apart from testing the normal operation of our system, we're interested in emulating situations where the network topology changes or breaks down:
- Host crash/restart. 
- Network partition/merge: the "split brain" scenario.

These two scenarios can be pretty cumbersome to set up, and are seldom exhaustively tested during the development phase. The aim of this post is to give the developer a tool for testing abnormal network situations, that can be integrated in the project from the start or applied to existing projects, without interfering with the project structure and dependencies.

The general idea is simple, and can be described in four steps. 

### Dockerize the nodes
For each node in the test scenario, create a Dockerfile and a Docker context. The Dockerfile defines a Docker image from the artifacts of the build process. The context, typically a directory on the file system, contains material that need to be added on top of the default artifact. For example, the default artifact for a certain node might be a zip file containing libraries, executables and a configuration file where the configurator needs to choose which network interface to use. The Docker context for this node would then contain that configuration file, with the network interface field filled out with a fixed address that will be assigned to that node on the virtual Docker network. The Dockerfile ensures that the file is added in the right location in the container, superimposed on the default configuration file.

### Create a build pipeline
The build pipeline builds project artifacts from source code, then builds the Docker images described in the previous step, perhaps tagging them suitably, e.g. `node-1`, `node-2` etc. The output from this pipeline is thus a Docker image (or tag) for each node in the scenarios, ready to be launched into a cluster. For a relatively simple project the pipeline would typically just be a script that can be launched from a terminal prompt.

>TIP: Building the Docker image for a large project may be time consuming. It might be possible to use the Docker image hierarchy to minimize build time overhead. For example, by building a root image from the project artifacts, then only adding configuration on the node images. This way, if the developer is experimenting with configuration changes, it will not be necessary to rebuild the root image. Add flags to bypass the corresponding build commands.

### Define the scenario setup
Docker Compose lets you define and run multiple containers on virtual networks with fixed, pre-defined IP addresses. Write a yaml or json file that defines your test setup, including virtual networks, environment variables, and all peripherals needed to test the application, like e.g. database containers with pre-defined data. The containers in the setup can then be individually stopped, restarted, or disconnected from the network with simple terminal commands.

### Design and implement tests
Last but not least, define and implement actual tests, that interact with the distributed system and generate output that might be evaluated. This part is naturally exceedingly reliant on the nature of the project, and there is not a lot to be said in general terms. Some projects will have obvious, clear cut test cases that are easy to define, implement and evaluate, others will not.

## The example project

[The example project](https://github.com/mxns/hazelcast-lab) is a Maven based Java project, utilising [Hazelcast](https://hazelcast.com/) for distributed storage. The application publishes a simple API over HTTP, with support for POST, GET and DELETE, backed by a distributed Hazelcast map. Each node in the project deploys the same JAR file, and is distinguished from the other nodes by configuration only.

Let's have look at how the four steps in the previous section are handled.

### Dockerization

The base image installs the application from the zip file produced by the local Maven build. To access the contents of the `target` directory, where the zip file resides after build, the Dockerfile needs to be located in the root directory of the main project. The child Dockerfiles, on the other hand, can be located anywhere as long as the Docker repo can be accessed, and their location can thus be separated from the main project. This is good, since we could let the test code constitute a separate project, instead of cluttering up the main project with Dockerfiles, code and dependencies. It also allows us to use version control to tag specific scenario setups, separated from the main project.

Here's the Dockerfile for the base image:

```
FROM java:8

ADD target/imap.zip /tmp/

RUN unzip -q /tmp/imap.zip -d /usr/local/

WORKDIR /usr/local/imap
```

Hazelcast node communication is configured via an XML file on the classpath. For TCP/IP communication, the IP address of all the nodes must be present in this file, which therefore needs to be superimposed in the node containers. The Dockerfile for the two nodes are identical, just overriding the configuration directory in the default distribution. Any file added to the `resources` directory in the build context will be added to the node images. For example, we can thus override the log4j configuration by just dropping a log4j.xml file in the `resources` directory and rebuild the images.

```
FROM imap

ADD resources /usr/local/imap/resources
```

### Build pipeline

The build pipeline is a simple bash script that cd's into project directories and issues various build commands. It outputs three Docker images: `imap`, `imap-node-1` and `imap-node-2`:

```
cd ${DEV_HOME}/hazelcast-lab
mvn clean install
docker build -t imap imap
docker build -t imap-node-1 imap-test/scenarios/multi-node/imap-node-1
docker build -t imap-node-2 imap-test/scenarios/multi-node/imap-node-2
``` 

Adding flags to skip phases of the process is left as an exercise for the reader.

### Test scenario

Our example test scenario contains the following items:

* Two Hazelcast based nodes
* Two virtual networks

One network is reserved for internal Hazelcast communication, while the application API is published on the other. The networks are named `nw1` and `nw2` in the YAML file, but running the setup with docker-compose will prefix the network names with the name of the directory where the file reside. 

### Test suite

We can use this simple example project to study how Hazelcast parameters influence the behaviour of the system under network stress. For example, try manipulating Heartbeat settings:

| Property Name | Default Value | Type  | Description |
| ------------- | ------------- | ----- |
| `hazelcast.heartbeat.interval.seconds` | 5 | int | Heartbeat send interval in seconds. |
| `hazelcast.max.no.heartbeat.seconds` | 300 | int | Maximum timeout of heartbeat in seconds for a member to assume it is dead. CAUTION: Setting this value too low may cause members to be evicted from the cluster when they are under heavy load: they will be unable to send heartbeat operations in time, so other members will assume that it is dead. |

 and see how the system behaves when nodes are stopped or disconnected from the network. See [Hazelcast System Properties](http://docs.hazelcast.org/docs/latest-development/manual/html/System_Properties/System_Properties_-_Member.html) for more parameters to play with.

Emulating a network partition and examining how the nodes behaves can be achieved thus:

```
docker network disconnect multinode_nw2 imap-node-1
docker network disconnect multinode_nw2 imap-node-2
curl -X POST -d 'key=animal' -d 'value=dog' http://127.0.0.1:18080/maps/my_map
curl -X POST -d 'key=animal' -d 'value=cat' http://127.0.0.1:28080/maps/my_map
docker network connect --ip 172.16.239.11 multinode_nw2 imap-node-1
docker network connect --ip 172.16.239.12 multinode_nw2 imap-node-2
curl -X GET http://127.0.0.1:18080/maps/my_map?key=animal
curl -X GET http://127.0.0.1:28080/maps/my_map?key=animal
```

How long does it take before the nodes are in sync? Which animal will be the final result?

## Summary

If your project is amenable to the outlined test method, it can be very useful. It can greatly help while developing micro services since it allows you to experiment and try things out quick and easy, in a realistic environment. It can provide some degree of certainty that basic networking functionality is intact across code changes, library updates etc. It allows you to examine how your system behaves during host restarts and network failures, without having to do the manual setup.

The approach might fail for a number of reasons:

 - Building and deploying the images takes too long. The time from changing a line of code to having a system a up and running should not exceed a few minutes, in order to maximize the usefulness of the test method. Try to design your project for short build times.

 - Designing and running tests that can actually be evaluated are too complicated or impossible. This might be due to the nature of the provided service, or because of bad design choices. A nice micro service should provide a testable API. 

- The distributed application relies heavily on technologies that cannot easily be reproduced in an offline development environment, e.g some of the cloud based services for the big cloud providers.


It might be argued that a testable system is in some sense a well designed system. Therefore, applying test methods such as this at an early stage in the development process, and thus making sure that modularity and high level integration testability is maintained from the start, could be helpful in making some preferable design choices.

