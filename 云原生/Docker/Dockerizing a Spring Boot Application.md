# Dockerizing a Spring Boot Application



## **1. Overview**



In this tutorial, we'll focus on how to dockerize a *Spring Boot Application* to run it in an isolated environment, a.k.a. *container*.

We'll learn how to create a composition of containers, which depend on each other and are linked against each other in a virtual private network. We'll also see how they can be managed together with single commands.

Let's start by creating a simple Spring Boot application that we'll then run in a lightweight base image running [*Alpine Linux*](https://hub.docker.com/_/alpine/).



## **2. Dockerize a Standalone Spring Boot Application**



As an example of an application that we can dockerize, we'll create a simple Spring Boot application, *docker-message-server,* that exposes one endpoint and returns a static message:

```java
@RestController
public class DockerMessageController {
    @GetMapping("/messages")
    public String getMessage() {
        return "Hello from Docker!";
    }
}
```

With a correctly configured Maven file, we can then create an executable jar file:

```shell
$> mvn clean package
```

Next, we'll start up the Spring Boot application:

```
$> java -jar target/docker-message-server-1.0.0.jar
```

Now we have a working Spring Boot application that we can access at *localhost:8888/messages.*

To dockerize the application, we first create a file named *Dockerfile* with the following content:

```
FROM openjdk:8-jdk-alpine
MAINTAINER baeldung.com
COPY target/docker-message-server-1.0.0.jar message-server-1.0.0.jar
ENTRYPOINT ["java","-jar","/message-server-1.0.0.jar"]
```

This file contains the following information:

- **FROM**: As the base for our image, we'll take the *Java*-enabled *Alpine Linux* created in the previous section.
- **MAINTAINER**: The maintainer of the image.
- **COPY**: We let *Docker* copy our jar file into the image.
- **ENTRYPOINT**: This will be the executable to start when the container is booting. We must define them as *JSON-Array* because we'll use an *ENTRYPOINT* in combination with a *CMD* for some application arguments.

To create an image from our *Dockerfile*, we have to run *‘docker build'* like before:

```
$> docker build --tag=message-server:latest .
```

Finally, we're able to run the container from our image:

```
$> docker run -p8887:8888 message-server:latest
```

This will start our application in Docker, and we can access it from the host machine at *localhost:8887/messages*. Here it's important to define the port mapping, which maps a port on the host (8887) to the port inside Docker (*8888*). This is the port we defined in the properties of the Spring Boot application.

Note: Port 8887 might not be available on the machine where we launch the container. In this case, the mapping might not work and we need to choose a port that's still available.

If we run the container in detached mode, we can inspect its details, stop it, and remove it with the following commands:

```shell
$> docker inspect message-server
$> docker stop message-server
$> docker rm message-server
```

### 2.1. Changing the Base Image

We can easily change the base image in order to use a different Java version. For example, if we want to use the Corretto distribution from Amazon, we can simply change the *Dockerfile*:

```shell
FROM amazoncorretto:11-alpine-jdk
MAINTAINER baeldung.com
COPY target/docker-message-server-1.0.0.jar message-server-1.0.0.jar
ENTRYPOINT ["java","-jar","/message-server-1.0.0.jar"]
```

Furthermore, we can use a custom base image. We'll look at how to do that later in this tutorial.



## **3. Dockerize Applications in a Composite**

*Docker* commands and *Dockerfiles* are particularly suitable for creating individual containers. However, if we want to *operate on a network of isolated applications*, the container management quickly becomes cluttered.

**To solve this, Docker provides a tool named Docker Compose.** This tool comes with its own build-file in *YAML* format, and is better suited for managing multiple containers. For instance, it's able to start or stop a composite of services in one command, or merges the logging output of multiple services together into one *pseudo-tty*.

### 3.1. The Second Spring Boot Application

Let's build an example of two applications running in different Docker containers. They will communicate with each other, and be presented as a “single unit” to the host system. As a simple example, we'll create a second Spring Boot application *docker-product-server*:

```java
@RestController
public class DockerProductController {
    @GetMapping("/products")
    public String getMessage() {
        return "A brand new product";
    }
}
```

We can build and start the application in the same way as our *message-server*.

### 3.2. The Docker Compose File

We can combine the configuration for both services in one file called *docker-compose.yml*:

```shell
version: '2'
services:
    message-server:
        container_name: message-server
        build:
            context: docker-message-server
            dockerfile: Dockerfile
        image: message-server:latest
        ports:
            - 18888:8888
        networks:
            - spring-cloud-network
    product-server:
        container_name: product-server
        build:
            context: docker-product-server
            dockerfile: Dockerfile
        image: product-server:latest
        ports:
            - 19999:9999
        networks:
            - spring-cloud-network
networks:
    spring-cloud-network:
        driver: bridge
```

- **version**: Specifies which format version should be used. This is a mandatory field. Here we use the newer version, whereas the *legacy format* is ‘1'.
- **services**: Each object in this key defines a *service*, a.k.a container. This section is mandatory.
  - **build**: If given, *docker-compose* is able to build an image from a *Dockerfile*
    - **context**: If given, it specifies the build-directory, where the *Dockerfile* is looked-up.
    - **dockerfile**: If given, it sets an alternate name for a *Dockerfile.*
  - **image**: Tells *Docker* which name it should give to the image when build-features are used. Otherwise, it's searching for this image in the library or *remote-registry.*
  - **networks**: This is the identifier of the named networks to use. A given *name-value* must be listed in the *networks* section.
- **networks**: In this section, we're specifying the *networks* available to our services*.* In this example, we let *docker-compose* create a named *network* of type *‘bridge'* for us. If the option *external* is set to *true*, it will use an existing one with the given name.

Before we continue, we'll check our build-file for syntax-errors:

```shell
$> docker-compose config
```

Then we can build our images, create the defined containers, and start it in one command:

```shell
$> docker-compose up --build
```

This will start up the *message-server* and *product-server* in one go.

To stop the containers, remove them from *Docker* and remove the connected *networks* from it. To do this, we can use the opposite command:

```shell
$> docker-compose down
```

For a more detailed introduction to *docker-compose,* we can read our article [Introduction to Docker Compose](https://www.baeldung.com/docker-compose).



### 3.3. Scaling Services

A nice feature of *docker-compose* is the **ability to scale services**. For example, we can tell *Docker* to run three containers for the *message-server* and two containers for the *product-server*.

However, for this to work properly, we have to remove the *container_name* from our *docker-compose.yml*, so *Docker* can choose names, and change the *exposed port configuration* to avoid clashes.

For the ports, we can tell Docker to map a range of ports on the host to one specific port inside Docker:

```shell
ports:
    - 18800-18888:8888
```

After that, we're able to scale our services like so (note that we're using a modified *yml-file*):

```shell
$> docker-compose --file docker-compose-scale.yml up -d --build --scale message-server=1 product-server=1
```

This command will spin up a single *message-server* and a single *product-server*.

To scale our services, we can run the following command:

```shell
$> docker-compose --file docker-compose-scale.yml up -d --build --scale message-server=3 product-server=2
```

This command will launch **two** additional message servers and **one** additional product server. The running containers won't be stopped.



## **4. Custom Base Image**

The base image (*openjdk:8-jdk-alpine*) we have used so far contained a distribution of the Alpine operating system with a JDK 8 already installed. Alternatively, we can build our own base image (based on Alpine or any other operating system).

To do so, we can use a *Dockerfile* with Alpine as a base image, and install the JDK of our choice:

```shell
FROM alpine:edge
MAINTAINER baeldung.com
RUN apk add --no-cache openjdk8
```

- **FROM**: The keyword *FROM* tells *Docker* to use a given image with its tag as build-base. If this image isn't in the local library, an online-search on *DockerHub*, or on any other configured remote-registry, is performed.
- **MAINTAINER**: A *MAINTAINER* is usually an email address, identifying the author of an image.
- **RUN**: With the *RUN* command, we're executing a shell command-line within the target system. Here we're utilizing *Alpine Linux's* package manager, *apk,* to install the *Java 8 OpenJDK.*

To finally build the image and store it in the local library, we have to run:

```shell
docker build --tag=alpine-java:base --rm=true .
```

**NOTICE:** The *–tag* option will give the image its name, and *–rm=true* will remove intermediate images after it's been built successfully. The last character in this shell command is a dot, acting as a build-directory argument.

Now we can use the created image instead of *openjdk:8-jdk-alpine*.



## 5. Buildpacks Support in Spring Boot 2.3

**Spring Boot 2.3 added support for** [**buildpacks**](https://buildpacks.io/). Put simply, instead of creating our own Dockerfile and building it using something like *docker build*, all we have to do is issue the following command:

```shell
$ mvn spring-boot:build-image
```

Similarly, in Gradle:

```shell
$ ./gradlew bootBuildImage
```

**For this to work, we need to have Docker installed and running.**

The main motivation behind buildpacks is to create the same deployment experience that some well-known cloud services, such as Heroku or Cloud Foundry, have been providing for a while. We just run the *build-image* goal, and then the platform itself takes care of building and deploying the artifact.

Moreover, it can help us change the way we're [building Docker images more effectively](https://www.baeldung.com/spring-boot-docker-images). Instead of applying the same change to lots of Dockerfiles in different projects, all we have to do is change or tune the buildpacks image builder.

**In addition to ease of use and better overall developer experience, it can also be more efficient.** For instance, the buildpacks approach will create a layered Docker image and uses the exploded version of the Jar file.

Let's look at what happens after we run the above command.

When we list the available docker images:

```shell
docker image ls -a
```

We see a line for the image we just created:

```shell
docker-message-server 1.0.0 b535b0cc0079
```

Here, the image name and version match the name and version we defined in the Maven or Gradle configuration file. The hash code is the short version of the image's hash.

Then to start our container, we can simply run:

```shell
docker run -it -p9099:8888 docker-message-server:1.0.0
```

As with our built image, we need to map the port to make our Spring Boot application accessible from outside Docker.



## **6. Conclusion**

In this article, we learned how to build custom *Docker* images, run a *Spring Boot Application* as a *Docker* container, and create containers with *docker-compose*.

For further reading about the build files, we refer to the official *Dockerfile reference* and the *docker-compose.yml reference*.

As usual, the source codes for this article can be found *on Github*.