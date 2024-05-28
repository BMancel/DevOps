
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


################################
Describe docker-compose.yml file
################################

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








