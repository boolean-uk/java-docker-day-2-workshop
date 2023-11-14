# Java Docker Day 2 Workshop

## Lesson Objectives

- Connect a Spring App in one container to a database in another container
- Setting up the application.yml file to connect to a containerised database
- Use docker-compose to simplify connections between multiple containers
- Use docker-compose to handle database credentials

## Connecting to a Local Database also in a Docker Image

We are going to make it so that our application running in its container connects to a Postgres database that runs in its own container, with both containers on the same machine. If we wanted to have our Postgres container on a separate machine, then as long as we started it correctly on that machine and connected up the correct port then we should be able to just connect to it as though it was running directly on the remote machine and listening on the given port (ie port 5432 the default Postgres port).

When we run our Spring Application in a container it sees the `localhost` as being inside the machine, so if we wanted to connect to another service running on the same host machine then we would need to know the IP address of that host in order to connect to it, but if the service is itself in a container then we can use a "trick" to connect them both together using docker itself. What we basically do is create a docker `network` that will allow us to connect together containers without worrying about what IP address they're running on, if  we needed to there are ways to lock down this functionality to prevent other users from gaining access to it or intercepting traffic on it.

Let's start by creating a new docker network which we'll use to let our Spring App container and our Database container talk to each other. In the terminal type:

```bash
docker network create daves-music-net
```

The `daves-music-net` part is the name of the docker network that has just been created. Run:

```bash
docker network ls
```

to see a list of all of the networks currently running. If you don't remove the network it will remain on your system even after rebooting.

First we're going to download and run an instance of `postgres` in its own container. Let's start by pulling the official `postgres` image from Docker Hub using:

```bash
docker pull postgres
```

Then we can run it, on our docker network using the following command:

```bash
docker run --name postgresdb --network daves-music-net -e POSTGRES_PASSWORD=mypassword -e POSTGRES_DATABASE=postgres -e POSTGRES_USER=postgres -d postgres
```

Check that it's running using `docker ps`. Now we're going to build the `jar` file for our Spring Application but with the following connection details in our `application.yml` file:

```yml
server:
  port: 4000
  error:
    include-message: always
    include-binding-errors: always
    include-stacktrace: never
    include-exception: false

spring:
  datasource:
    url: jdbc:postgresql://postgresdb:5432/postgres
    username: postgres
    password: mypassword
    max-active: 3
    max-idle: 3
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
      show_sql: true
```

The database url uses the name of our postgres docker container `postgresdb`, the database name at the end of the url and the username are the ones we specified in the command we ran the database with above as is the password. To build the application we will need to make it ignore the built in tests, as otherwise the build will fail. Open the terminal in the top level of the Spring Application and run the following command which ignores any tests and generates the required `jar` file.

```bash
./gradlew build -x test
```

Copy the resulting `jar` file into the folder with a Dockerfile that has the same basic contents as last time and then build the image with:

```bash
docker build -t daves-music-box .
```

or your equivalent. If that succeeds then you should be able to run it in the same docker network as the database with:

```bash
docker run --network daves-music-net --name daves-music-box -p 4000:4000 -d daves-music-box
```

You should then be able to fire up Insomnia and interact with your endpoints using that. Unless you've created records in the database tables there won't be anything in there. You should be able to stop and start the containers without too much problem. If you delete the instance of the database one and then rerun it using the same commands as previously the data will not persist between runs. You can overcome this by connecting the database to a `volume` to persist data but we aren't going to go into that here.

Until you prune the container instance, you can always restart it using `docker start container-instance-name` so if I had stopped the Spring Applicatation container I could restart it using `docker start daves-music-box` and it would restart with the same settings as we previously gave it. If I called prune on any stopped containers I would need to redo the whole `run` command as shown above and any data in the database would  be lost if the database container was the one which was pruned and then rerun.

## Using docker-compose to Simplify Things

Although we can now start multiple containers and make them talk to each other, there is a certain amount of prior knowledge needed in order to do so, we can instead use a command called `docker-compose` to tie everything together and make them all run simultaneously from a single command.

We're going to use a Docker Compose file to deal with all of the database connection shenanigans this time, so delete the database part (everythiong from the word `spring` down) from the `application.yml` file and then rebuild the spring app, using `./gradlew build -x test`. Copy the `jar` file across and build a new image from it (I'm calling mine `music-box`).

Now create a new file called `docker-compose.yml` somewhere, I put this in the same folder where my `jar` file and `Dockerfile` are but just to keep things together, there is no need for them to be in the same place.

Inside the `docker-compose.yml` file we are going to detail the two images we want to talk to each other as well as the connection details we previously had inside `application.yml`. Each image is known as a `service` and we need to specify various aspects of how the two services will communicate with each other. Here's a working one:

```yml
services:
  app:
    image: 'music-box:latest'
    container_name: app
    depends_on:
      - db
    ports:
      - '4000:4000'
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/mypostgresuser
      - SPRING_DATASOURCE_USERNAME=mypostgresuser
      - SPRING_DATASOURCE_PASSWORD=mypostgrespassword
      - SPRING_JPA_HIBERNATE_DDL_AUTO=update

  db:
    image: 'postgres:latest'
    container_name: db
    environment:
      - POSTGRES_USER=mypostgresuser
      - POSTGRES_DATABASE=mypostgresuser
      - POSTGRES_PASSWORD=mypostgrespassword
```

I'm unclear whether the service name and the container name need to match. It appears that the database name at the end of the URL and in the database details needs to match the user name for Postgres. The `depends_on` setting tells docker to spin up the database one first and then allow the application to talk to it. In the background this will create a default docker network for the containers to talk to each other over.

Once everything;s set up, use:

```bash
docker-compose up -d
```

to start it running, and detach the process from the terminal.

Going to [http://localhost:4000](http://localhost:4000) on my local machine allows me to interact with the endpoints as normal. If you had a React frontend that was also in a container then you would talk to that and it in turn would talk to the API.

Things you need to investigate further are:

- How to secure this sort of set up?
- How a React container will know which port of the API to talk to?
- How to use volumes to persist the data in the database?
