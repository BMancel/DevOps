
# Readme

Here is the repo of the 3 TPs doing during DevOps, from 05-27-2024 to 05-30-2024


## Author

- [@BMancel](https://github.com/BMancel)


## TP1 - Docker

### 1-1 Document your database container essentials: commands and Dockerfile

The goal of this TP is to create a 3-tiers application :
- HTTP server
- Backend API
- Database

#### **Database**

Creation of a Dockerfile :

```
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd
```

Creation of an image from the Dockerfile :

```
docker build -t mypostgres:2 .
```

Creation of a volume :

```
docker volume create postgres_data
```

Creation of a network :

```
docker network create app-network
```

Create and launch the admin container. This allows communication with the PostgreSQL database on host port 8090 :

```
docker run -p "8090:8080" --net=app-network --name=adminer -d admine
```

Save the data in the volume and create the mypostgres container :

```
docker run -p 5432:5432 --name mypostgres --network app-network -d -v C:\Users\mance\Desktop\Etudes\4A\S8\DevOps\Day1\CreateScheme.sql:/docker-entrypoint-initdb.d/CreateScheme.sql -v C:\Users\mance\Desktop\Etudes\4A\S8\DevOps\Day1\InsertData.sql:/docker-entrypoint-initdb.d/InsertData.sql -v postgres_data:/var/lib/postgresql/data mypostgres:2
```

the data inserted into the mypostgres container :

- Creation of the tables :
```
CREATE TABLE public.departments
(
 id      SERIAL      PRIMARY KEY,
 name    VARCHAR(20) NOT NULL
);

CREATE TABLE public.students
(
 id              SERIAL      PRIMARY KEY,
 department_id   INT         NOT NULL REFERENCES departments (id),
 first_name      VARCHAR(20) NOT NULL,
 last_name       VARCHAR(20) NOT NULL
);
```

- Filling the tables :
```
INSERT INTO departments (name) VALUES ('IRC');
INSERT INTO departments (name) VALUES ('ETI');
INSERT INTO departments (name) VALUES ('CGP');


INSERT INTO students (department_id, first_name, last_name) VALUES (1, 'Eli', 'Copter');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Emma', 'Carena');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Jack', 'Uzzi');
INSERT INTO students (department_id, first_name, last_name) VALUES (3, 'Aude', 'Javel');
```
#### Screenshot

Visualization of a table from the mypostgres container on port 8090 using the adminer container

![image](https://github.com/BMancel/DevOps/assets/150273847/9b877b7a-42a5-4e6f-8255-6f14cd7e8064)

#### **Backend API**

For the Backend, we have the dockerfile :

We use a multistage build: A multistage build allows us to use only what is necessary, and therefore to reduce the size of the image. In addition to saving space, a lightweight image deploys more quickly.

```
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

First, we compile our project. To do this, we start from a maven image, we define its working directory, copy the pom.xml file and the src folder into the container. Finally, maven compiles the code (without testing).

Then, we start from an image of Amazon Corretto, a jre compatible with our jdk, to execute the code once compiled. We copy myapp constructed above to our directory. Finally, we specify the command to execute when starting the container: we execute the myapp.jar file with Java.

#### **Http server**

Creation of a Dockerfile which will later allow the creation of the myhttp container, which will contain the modified httpd.conf file:

```
FROM httpd:2.4

COPY index.html /usr/local/apache2/htdocs/
COPY httpd.conf /usr/local/apache2/conf/httpd.conf
```

The modification adds a reverse proxy to our application.

The modification inside the httpd.conf file :

```
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://springboot:8080/
ProxyPassReverse / http://springboot:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

#### Screenshot

Visualization of the api when it fetches data via the path /departments/IRC/students :

![image](https://github.com/BMancel/DevOps/assets/150273847/6ac12193-9b5d-4470-8b4d-8dcd079a5452)

#### **Link application**

Docker Compose makes it easier to manage multi-container applications, making the deployment and development process faster and more coherent. It allows to launch entire development environments with a single command.

I have Docker desktop on my machine, so docker-compose is already installed.

creation of a docker-compose.yml file :

```
version: '3.7'

services:
    backend:
        build:
            context: ./backend
        networks:
            - my-network
        depends_on:
            - database
        container_name : backend-container

    database:
        build:
            context: ./database
        networks:
            - my-network
        volumes:
            - my_volume:/var/lib/postgresql/data
        container_name : database-container

    httpd:
        build:
            context: ./frontend
        ports:
            - "80:80"
        networks:
            - my-network
        depends_on:
            - backend
        container_name : frontend-container

networks:
    my-network:

volumes:
    my_volume:
```

Shape of my directory :

![image](https://github.com/BMancel/DevOps/assets/150273847/ce4b4d7d-2990-465a-80c2-0ca027d995b5)

We can create and start the docker-compose with the command :

```
docker-compose up -d --build
```

If everything works correctly :

![image](https://github.com/BMancel/DevOps/assets/150273847/b1899db8-4c30-4424-a6fc-6267cd1d548d)

![image](https://github.com/BMancel/DevOps/assets/150273847/746bb073-1f61-4a9b-be01-fcb7d4a73533)

The docker-compose.yml file builds the images and containers of the 3-tiers application, the backend, the database and the http server. We address a common network to each tier so that the different parties can communicate with each other and we provide them with a common volume to recover data if the containers are deleted. the 'depends_on' condition allows to create the three parties in the correct order: first the database, then the backend and finally the http server. Finally, container_name allows to name containers to make them easier to find on Docker Desktop.

#### **Publish**

Connection to the docker hub with the command :

```
docker login
```

We tag our images and push them to docker hub :

```
docker tag docker_compose-database unptitcurieux/my-database:1.0
docker tag docker_compose-backend unptitcurieux/my-backend:1.0
docker tag docker_compose-httpd unptitcurieux/my-httpd:1.0
```
```
docker push unptitcurieux/my-database:1.0
docker push unptitcurieux/my-backend:1.0
docker push unptitcurieux/my-httpd:1.0
```

On Docker Hub :

![image](https://github.com/BMancel/DevOps/assets/150273847/b2731a34-4699-4431-b0dd-f3f7dc0d4e4b)


## TP2 - Github Actions

#### **CI (Continuous Integration)**

Inside the pom.xml file of the backend folder, we can see the following dependencies :

```
<dependency>
	<groupId>org.testcontainers</groupId>
	<artifactId>testcontainers</artifactId>
	<version>${testcontainers.version}</version>
	<scope>test</scope>
</dependency>
<dependency>
	<groupId>org.testcontainers</groupId>
	<artifactId>jdbc</artifactId>
	<version>${testcontainers.version}</version>
	<scope>test</scope>
</dependency>
<dependency>
	<groupId>org.testcontainers</groupId>
	<artifactId>postgresql</artifactId>
	<version>${testcontainers.version}</version>
	<scope>test</scope>
</dependency>
<dependency>
```
Testcontainers is a Java library that simplifies the process of creating and managing containerized environments for testing purposes. It allows developers to run their tests with dependencies like databases, message queues, web servers, and other services in isolated Docker containers. This ensures that the tests are reproducible, reliable, and do not interfere with the local development environment.

Architecture of the pipeline (in the main.yml file, in worklows) :
```
name: CI devops 2024
on:
  #to begin you want to launch this job in main and develop
  push:
    branches:
      - main
      - develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'  # Specify the JDK distribution
          java-version: '17'

     #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean verify --file docker_compose/backend/pom.xml
```

![image](https://github.com/BMancel/DevOps/assets/150273847/1167d497-87b4-4e84-bb07-ee32e462c867)


The workflow is triggered when I push or pull on my repo. It runs backend tests on an Ubuntu environment consisting of 3 steps: 
It starts by retrieving the source code from the Github repo, 
```
- uses: actions/checkout@v2.5.0
```
then installs and configures a jdk, 
```
- name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'  # Specify the JDK distribution
          java-version: '17'
```
and finally compiles and tests the maven project located in the pom.xml file.
```
- name: Build and test with Maven
        run: mvn clean verify --file docker_compose/backend/pom.xml
```

#### **CD (Continuous Delivery)**

Final main.yml :

```
name: CI devops 2024
on:
  #to begin you want to launch this job in main and develop
  push:
    branches:
      - main
      - develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'  # Specify the JDK distribution
          java-version: '17'

     #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean -B verify sonar:sonar -Dsonar.projectKey=bmancel_mancel-benjamin -Dsonar.organization=bmancel -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} -f ./docker_compose/backend/pom.xml

  # define job to build and publish docker image
  build-and-push-docker-image: 
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04

    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./docker_compose/backend
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/my-backend:latest

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./docker_compose/database
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/my-database:latest

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./docker_compose/frontend
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/my-httpd:latest 
```

![image](https://github.com/BMancel/DevOps/assets/150273847/07640d1d-17a1-4173-83e4-108a961f8e18)


quality gate : 

```
run: mvn clean -B verify sonar:sonar -Dsonar.projectKey=bmancel_mancel-benjamin -Dsonar.organization=bmancel -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  -f ./docker_compose/backend/pom.xml
```

A Quality Gate is a set of conditions defined to assess whether a project meets minimum quality criteria (e.g. no blocking bugs, minimum test coverage, etc.). If the project does not meet these conditions, the Quality Gate fails, indicating that the code is not ready to be merged or deployed.

Here :

`mvn clean` : Cleans the project by removing files generated during previous compilations.

`clean` : Executes a complete build lifecycle, including compilation and unit testing. This phase ensures that the project is valid and well structured.

`sonar:sonar` : Evaluate code quality, detect bugs, vulnerabilities and bad coding practices.

![image](https://github.com/BMancel/DevOps/assets/150273847/615572aa-3e04-4615-9577-f388c1d1c13b)

#### **Bonus : split pipelines**

In this step I have to separate my jobs into 2 different workflows :

 - test-backend.yml :

```
name: Test Backend
 
on:
  push:
    branches:
      - main
      - develop
 
jobs:
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v2.5.0
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '17'
 
      - name: Build and test with Maven
        run: mvn clean -B verify sonar:sonar -Dsonar.projectKey=bmancel_mancel-benjamin -Dsonar.organization=bmancel -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  -f ./docker_compose/backend/pom.xml
```

 - build-and-push-docker-image.yml :

```
name: Build and Push Docker Image
 
on:
  workflow_run:
    workflows: ["Test Backend"]
    types:
      - completed
    branches: main
 
jobs:
  build-and-push-docker-image:
    runs-on: ubuntu-22.04
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0
 
      - name: Login to Docker Hub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}
 
      - name: Build and push backend image
        uses: docker/build-push-action@v2
        with:
          context: ./docker_compose/backend
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/my-backend:latest
 
      - name: Build and push database image
        uses: docker/build-push-action@v2
        with:
          context: ./docker_compose/database
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/my-database:latest
 
      - name: Build and push frontend image
        uses: docker/build-push-action@v2
        with:
          context: ./docker_compose/frontend
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/my-database:latest
```

![image](https://github.com/BMancel/DevOps/assets/150273847/be964b3a-876a-4d1f-a7eb-fbd1e8ee749d)




