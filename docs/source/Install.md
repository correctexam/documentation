
# AMD64

## Build

###  Backend

If you want to build it manually, you can just run the following command: 

```bash
git clone https://github.com/correctexam/corrigeExamBack
cd corrigeExamBack
mvn -B package --file pom.xml -Pnative
docker build -f src/main/docker/Dockerfile.native -t barais/correctexam-back:manifest-amd64 --build-arg ARCH=amd64/  .
```



The backend is also built automatically using github action. You can have access to the backend image within [docker hub](https://hub.docker.com/repository/docker/barais/grade-scope-istic)

**OR** 

```bash
#if you install the quarkus cli
git clone https://github.com/correctexam/corrigeExamBack
cd corrigeExamBack
quarkus build --native --no-tests -Dquarkus.native.container-build=true  
docker build -f src/main/docker/Dockerfile.native -t barais/correctexam-back:manifest-amd64 --build-arg ARCH=amd64/  .
```


###  Front

**Without docker**
Just clone the project

:warning:update webpack/environment.js with your domain name.


```bash
# require nodejs v16 you can install it using nvm (https://github.com/nvm-sh/nvm)
git clone https://github.com/correctexam/corrigeExamFront
cd corrigeExamFront
# update webpack/environment.js with your domain names
npm install 
npm run webapp:build:prod
## You can be inspired by webapp:build:prodgithubpage task if you want to manage a prfix for your webapp. 
## if you want to deploy on github page, you can be inspired by the provided github action
```

**Using docker**

To build the front, we provide a simple docker file. 

:warning:update webpack/environment.js with your domain name.


```bash
git clone https://github.com/correctexam/corrigeExamFront
cd corrigeExamFront
# update webpack/environment.js with your domain name
sudo docker build -f src/main/docker/Dockerfile -t barais/correctexam-front:manifest-amd64 --build-arg ARCH=amd64/ .
# OR 
sudo docker buildx build  -f src/main/docker/Dockerfile --push --platform linux/arm64,linux/amd64  --tag barais/correctexam-front .

```

You will obtain a nginx with only the js, html and js. You have to mount the configuration if you want to manage proxy to the backend routes. I would prefer to use bunkerized nginx for the routing. 



## Deploy everything on your own infrastructure



```yaml
version: '2'
services:
  correctexam-back:
    image: barais/correctexam-back:manifest-amd64
    volumes:
# Path for 
      - /tmp/files:/tmp/files:rw 
    restart: always
    ports:
      - 8080:8080
# All quarkus configuration parameters (knobs) could be override through command line. You could also use different options to update the configuration parameters.  https://quarkus.io/guides/config-reference
    command: ./application -Dquarkus.http.host=0.0.0.0 -Dquarkus.datasource.username=root -Dquarkus.datasource.password='' -Dquarkus.datasource.jdbc.url=jdbc:mysql://correctexam-mysql:3306/correctexam?useUnicode=true&characterEncoding=utf8&useSSL=false -Dquarkus.http.cors=true -Dquarkus.http.cors.origins=https://correctexam.github.io -Dquarkus.http.cors.methods=GET,PUT,POST,DELETE,PATCH,OPTIONS -Dquarkus.http.cors.headers=accept,origin,authorization,content-type,x-requested-with -Dquarkus.http.cors.exposed-headers=Content-Disposition -Dquarkus.http.cors.access-control-max-age=24H -Dquarkus.mailer.from=olivier.barais@univ-rennes1.fr -Dquarkus.mailer.host=partage.univ-rennes1.fr -Dquarkus.mailer.port=587 -Dquarkus.mailer.ssl=false -Dquarkus.mailer.username=olivier.barais@univ-rennes1.fr -Dquarkus.mailer.password=TOCHANGE -Djhipster.mail.base-url=https://correctexam.github.io/corrigeExamFront
  correctexam-mysql:
    image: mysql:8.0.20
    volumes:
      - ./../resources/db/migration/:/docker-entrypoint-initdb.d
    environment:
      - MYSQL_USER=root
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
      - MYSQL_DATABASE=correctexam
    command: mysqld --lower_case_table_names=1 --skip-ssl --character_set_server=utf8mb4 --explicit_defaults_for_timestamp
#    ports:
#      - 3308:3306
  front:
    image: barais/correctexam-front:manifest-amd64
#    ports:
#      - 90:80
    volumes:
      -  ./exampleconf/exam.conf:/etc/nginx/conf.d/exam.conf
      -  ./exampleconf/nginx.conf:/etc/nginx/nginx.conf:ro
```


**exam.conf** and **nginx.conf** could be something like that  (you have to update the server name)

**exam.conf**


```nginx
server {
    listen       80;
    listen  [::]:80;
    # server name to change based on your own domain name for your front
    server_name  correctexam.barais.fr;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location /api {
        proxy_pass http://correctexam-back:8080/api;
        proxy_set_header Host $http_host;

    }

    location /api {
        include proxy_params;
    	proxy_pass http://correctexam-back:8080/api;
    }

    location /management {
        include proxy_params;
        proxy_pass http://correctexam-back:8080/management;
    }

    location /swagger-ui {
        include proxy_params;
        proxy_pass http://correctexam-back:8080/swagger-ui;
    }

    location /v3/api-docs {
        include proxy_params;
        proxy_pass http://correctexam-back:8080/v3/api-docs;
    }

    location /auth {
	include proxy_params;
        proxy_pass http://correctexam-back:8080/auth;

    }

    location /health {
        include proxy_params;
        proxy_pass http://correctexam-back:8080/health;
    }

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html?$args;

    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

}
```

**nginx.conf**

```nginx
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 1000s;
	types_hash_max_size 2048;


    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    gzip  on;
	client_max_body_size 900M;
	client_body_buffer_size     900M;

    include /etc/nginx/conf.d/*.conf;
}
```

## Or Deploy the database and the backend on your own infrastrcture and the frontend on a CDN

If you want to deploy the database and the backend infrastrcture on your own infrastrcuture and deploy the frontend on the CDN. You must have to manage properly your CORS authorization within your CDN and withing your backend. If you use github page public site, Pages allows CORS (access-control-allow-origin header is set to *). For the backend, you can use the quarkus properties *-Dquarkus.http.cors=true -Dquarkus.http.cors.origins=https://correctexam.github.io -Dquarkus.http.cors.methods=GET,PUT,POST,DELETE,PATCH,OPTIONS* to manage your cors. Please update the docker-compose descriptor accordingly. 

To start your backend and your frontend, I will propose the skeleton of a docker-compose. Please update it for your needs. 
On top of that, you can configured bunkerized nginx to automatically setup your security and let's encrypt certificate. This docker compose automatically setup the database and the backend. You must update the ./../resources/db/migration/ sql file that will be executed when the database will start. It populates the database with initial data. 


```yaml
version: '2'
services:
  correctexam-back
    image: barais/correctexam-back:manifest-amd64
    volumes:
# Path for 
      - /tmp/files:/tmp/files:rw 
    restart: always
    ports:
      - 8080:8080
# All quarkus configuration parameters (knobs) could be override through command line. You could also use different options to update the configuration parameters.  https://quarkus.io/guides/config-reference
    command: ./application -Dquarkus.http.host=0.0.0.0 -Dquarkus.datasource.username=root -Dquarkus.datasource.password='' -Dquarkus.datasource.jdbc.url=jdbc:mysql://correctexam-mysql:3306/correctexam?useUnicode=true&characterEncoding=utf8&useSSL=false -Dquarkus.http.cors=true -Dquarkus.http.cors.origins=https://correctexam.github.io -Dquarkus.http.cors.methods=GET,PUT,POST,DELETE,PATCH,OPTIONS -Dquarkus.http.cors.headers=accept,origin,authorization,content-type,x-requested-with -Dquarkus.http.cors.exposed-headers=Content-Disposition -Dquarkus.http.cors.access-control-max-age=24H -Dquarkus.mailer.from=olivier.barais@univ-rennes1.fr -Dquarkus.mailer.host=partage.univ-rennes1.fr -Dquarkus.mailer.port=587 -Dquarkus.mailer.ssl=false -Dquarkus.mailer.username=olivier.barais@univ-rennes1.fr -Dquarkus.mailer.password=TOCHANGE -Djhipster.mail.base-url=https://correctexam.github.io/corrigeExamFront
  correctexam-mysql:
    image: mysql:8.0.20
    volumes:
      - ./../resources/db/migration/:/docker-entrypoint-initdb.d
    environment:
      - MYSQL_USER=root
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
      - MYSQL_DATABASE=correctexam
    command: mysqld --lower_case_table_names=1 --skip-ssl --character_set_server=utf8mb4 --explicit_defaults_for_timestamp
    ports:
      - 3308:3306
```

Before building the frontend for your CDN, do not forget to update **webpack/environment.js** with your domain names. 

#  Build and deploy on raspberry PI (arm64)


##  Install support of cross compile on your machine


```bash 
sudo apt-get install qemu binfmt-support qemu-user-static
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes # This step will execute the registering scripts
docker run --rm -t arm64v8/ubuntu uname -m 
```

##  Build

### Build the backend

Create a docker file for quarkus to cross compile on your machine for arm64

```dockerfile
FROM ghcr.io/graalvm/graalvm-ce:ol8-java11@sha256:dc9effae9a92d50e0a173f1cb8113409a4b6d7fb0c44fcf2195f0e03d6161bc5 AS build
RUN gu install native-image
WORKDIR /project
VOLUME ["/project"]
ENTRYPOINT ["native-image"] 
```

Build the image

```bash 
docker build -f src/main/docker/Dockerfile.build.aarch64 -t barais/quarkus-build-aarch64 .
```

Next build your executable

```bash
./mvnw clean package -Pnative -Pprod -Dquarkus.package.type=native -DskipTests=true  -Dquarkus.native.container-build=true -Dquarkus.native.builder-image=barais/quarkus-build-aarch64:latest
```

**OR** build it using quarkus cli

```bash
quarkus build --native --no-tests -Dquarkus.native.container-build=true -Dquarkus.native.builder-image=barais/quarkus-build-aarch64
```

When the compilation will over (could take a long time ;), you can build the final images with the binary.

**Create the base image for your backend**



```dockerfile
FROM registry.access.redhat.com/ubi8/ubi-minimal@sha256:c6592eb9cdd7ea7fa43beddf507ca2a8c2127f13ef66d49baea2fd28e37f62ba
WORKDIR /work/
RUN chown 1001 /work \
    && chmod "g+rwX" /work \
    && chown 1001:root /work
#COPY --chown=1001:root target/*-runner /work/application
# COPY --chown=1001:root ./src/main/resources/db/migration/ /work/migration
COPY target/*-runner /work/application
RUN chmod 775 /work
EXPOSE 8080
USER 1001
CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
```



```bash
docker build -f src/main/docker/Dockerfile.arm64 -t barais/correctexam-back:manifest-arm64v8 --build-arg ARCH=arm64v8/ .
```

###  Build the frontend

Clone the frontend repository. 

:warning:update webpack/environment.js with your domain name.


```bash
git clone https://github.com/correctexam/corrigeExamFront
cd corrigeExamFront
# update webpack/environment.js with your domain name
sudo docker build -f src/main/docker/Dockerfile.arm64 -t barais/correctexam-front::manifest-arm64v8 --build-arg ARCH=arm64v8/ .
# OR using buildx
sudo docker buildx build  -f src/main/docker/Dockerfile --push --platform linux/arm64,linux/amd64  --tag barais/correctexam-front .
```

You will obtain a nginx with only the js, html and js. You have to mount the configuration if you want to manage proxy to the backend routes. I would prefer to use bunkerized nginx for the routing. 



## Deploy on your raspberry 4

You can push your built image on dockerhub (update docker image within the docker compose) and just deploy the docker compose on your own raspberry with the nginx configuration files.



```yaml
version: '2'
services:
  correctexam-back:
    image: barais/correctexam-back::manifest-arm64v8
    volumes:
# Path for 
      - /tmp/files:/tmp/files:rw 
    restart: always
    ports:
      - 8080:8080
# All quarkus configuration parameters (knobs) could be override through command line. You could also use different options to update the configuration parameters.  https://quarkus.io/guides/config-reference
    command: ./application -Dquarkus.http.host=0.0.0.0 -Dquarkus.datasource.username=root -Dquarkus.datasource.password='' -Dquarkus.datasource.jdbc.url=jdbc:mysql://correctexam-mysql:3306/correctexam?useUnicode=true&characterEncoding=utf8&useSSL=false -Dquarkus.http.cors=true -Dquarkus.http.cors.origins=https://correctexam.github.io -Dquarkus.http.cors.methods=GET,PUT,POST,DELETE,PATCH,OPTIONS -Dquarkus.http.cors.headers=accept,origin,authorization,content-type,x-requested-with -Dquarkus.http.cors.exposed-headers=Content-Disposition -Dquarkus.http.cors.access-control-max-age=24H -Dquarkus.mailer.from=olivier.barais@univ-rennes1.fr -Dquarkus.mailer.host=partage.univ-rennes1.fr -Dquarkus.mailer.port=587 -Dquarkus.mailer.ssl=false -Dquarkus.mailer.username=olivier.barais@univ-rennes1.fr -Dquarkus.mailer.password=TOCHANGE -Djhipster.mail.base-url=https://correctexam.github.io/corrigeExamFront
  correctexam-mysql:
    image: mysql:8.0.20
    volumes:
      - ./../resources/db/migration/:/docker-entrypoint-initdb.d
    environment:
      - MYSQL_USER=root
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
      - MYSQL_DATABASE=correctexam
    command: mysqld --lower_case_table_names=1 --skip-ssl --character_set_server=utf8mb4 --explicit_defaults_for_timestamp
#    ports:
#      - 3308:3306
  front:
    image: barais/correctexam-front:manifest-arm64v8
#    ports:
#      - 90:80
    volumes:
      -  ./exampleconf/exam.conf:/etc/nginx/conf.d/exam.conf
      -  ./exampleconf/nginx.conf:/etc/nginx/nginx.conf:ro
```


**exam.conf** and **nginx.conf** could be something like that (you have to update the server name)

**exam.conf**


```nginx
server {
    listen       80;
    listen  [::]:80;
    # server name to change based on your own domain name for your front
    server_name  correctexam.barais.fr;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location /api {
        proxy_pass http://correctexam-back:8080/api;
        proxy_set_header Host $http_host;

    }

    location /api {
        include proxy_params;
    	proxy_pass http://correctexam-back:8080/api;
    }

    location /management {
        include proxy_params;
        proxy_pass http://correctexam-back:8080/management;
    }

    location /swagger-ui {
        include proxy_params;
        proxy_pass http://correctexam-back:8080/swagger-ui;
    }

    location /v3/api-docs {
        include proxy_params;
        proxy_pass http://correctexam-back:8080/v3/api-docs;
    }

    location /auth {
	include proxy_params;
        proxy_pass http://correctexam-back:8080/auth;

    }

    location /health {
        include proxy_params;
        proxy_pass http://correctexam-back:8080/health;
    }

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html?$args;

    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

}
```

**nginx.conf**

```nginx
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 1000s;
	types_hash_max_size 2048;


    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    gzip  on;
	client_max_body_size 900M;
	client_body_buffer_size     900M;

    include /etc/nginx/conf.d/*.conf;
}
```

# Create a release on docker hub

Next for the front you can use dockerX or create your docker image for the different targeted architecture. 

```bash
docker manifest create \
barais/correctexam-front:manifest-v1 \
--amend barais/correctexam-front:manifest-amd64 \
--amend barais/correctexam-front:manifest-arm64v8
docker manifest push barais/correctexam-front:manifest-v1
```

For the back, create your docker image for the different targeted architecture. 

```bash
docker manifest create \
barais/correctexam-back:manifest-v1 \
--amend barais/correctexam-back:manifest-amd64 \
--amend barais/correctexam-back:manifest-arm64v8
docker manifest push barais/correctexam-back:manifest-v1

```
