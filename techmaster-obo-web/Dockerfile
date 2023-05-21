FROM openjdk:20-oraclelinux8
COPY /target/obo-0.0.1-SNAPSHOT.jar /target/obo-0.0.1-SNAPSHOT.jar
COPY /src/main/resources/ /src/main/resources/

ENTRYPOINT ["java", "-jar", "target/obo-0.0.1-SNAPSHOT.jar"]