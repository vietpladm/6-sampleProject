## Build and deploy an application using Java Maven and mysql database in K8S environment.
(Application here is a website selling sport shoes.)

Source code of the website was cloned from: https://github.com/liamhubian/techmaster-obo-web

Target of the lab:

- Step 1: Create the mysql database docker container.
- Step 2: Create maven docker container with the current source code. 
- Step 3: Build the image and push it to dockerhub.
- Step 4: Deploy the application to K8S with the docker image built above. (eg. obo-web deployment)
- Step 5: Forward the obo-web deployment to the host machine. Try to access the website from http://localhost:8080

# Step 1: Create mysql docker container:

Create the mysql container with the default password as 123:
```sh
docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123 mysql
```
Use the command Docker ps -a to see the mysql container's name
eg. the mysql container name is "golden_view"

Copy file database obo.sql from the current source code folder (techmaster-obo-web) into the  mysql container
```sh
docker cp obo.sql golden_view:/
```
Shell into the mysql container to import the database obo.sql

```sh
docker exec -ti golden_view bash
#Access the mysql database with password as 123
bash-4.4# mysql -u root -p
#import the database from the copied file obo.sql
mysql> create database obo;
mysql> exit
bash-4.4# mysql -u root -p obo < obo.sql
```

# Step 2: Create the Maven docker container and mount the current source code folder as volume.
```sh
Docker run -ti -v $PWD:/obo-web maven bash
cd obo-web
mvn install
exit
```

# Step 3: Build the image and push it to dockerhub.

Create the Dockerfile with the content as below:
```
FROM openjdk:20-oraclelinux8
COPY /target/obo-0.0.1-SNAPSHOT.jar /target/obo-0.0.1-SNAPSHOT.jar
COPY /src/main/resources/ /src/main/resources/

ENTRYPOINT ["java", "-jar", "target/obo-0.0.1-SNAPSHOT.jar"]
```

Run command from CLI to build the docker image, then push it to dockerhub.
```
docker build -t vietpl/obo-web:1.0 .
docker push vietpl/obo-web:1.0
```

# Step 4: Deploy the application to K8S with the docker image built above. (eg. obo-web deployment)

Fristly first, we need to find the IP address of the first virtual ethernet adapter of our Ubuntu virtual machine. Later we will use it to configure the variable "spring.datasource.url" of Spring setting. 

Command:
```sh
ip a
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:de:68:73 brd ff:ff:ff:ff:ff:ff
    inet 172.21.161.250/20 brd 172.21.175.255 scope global eth0
```

Now we will create a configmap configuration to inject the value of spring.datasource.url into the application to match with the eth0 ip address 172.21.161.250

```
kubectl create configmap spring-config --dry-run=client --from-literal=spring.datasource.url="jdbc:mysql://172.21.161.250:3306/obo?createDatabaseIfNotExist=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC" -o yaml > templates/config-map.yaml
```
Double check the output YAML file of the ConfigMap

```sh
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: spring-config
data:
  spring.datasource.url: "jdbc:mysql://172.21.161.250:3306/obo?createDatabaseIfNotExist=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC"
```

Create the obo-web deployment configuration file and save to /templates/obo-web-deploy.yaml 
```sh
k create deployment obo-web --image vietpl/obo-web:1.0 -o yaml --dry-run=client > templates/obo-web-deploy.yaml
```
Open and modify the obo-web-deploy to match with the docker image we already built above and the configmap above.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: obo-web
  name: obo-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: obo-web
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: obo-web
    spec:
      containers:
      - image: vietpl/obo-web:1.6
        name: obo-web
        resources: {}
        envFrom:
        - configMapRef:
            name: spring-config
status: {}
```

# Step 5: Forward the obo-web deployment to the host machine. Try to access the website from http://localhost:8080

From CLI, run the below command to forward the port of obo-web-deployment
```
k port-forward deploy/obo-web --address 0.0.0.0 8080:8080
```

From the host machine, access the applciation successfully from browser with address 

```
http://localhost:8080
```

