
# Applying Multi Stage Build Method

### Improvements with Multi Stage Build Application

1) Reason 
 - Previously, even minor changes required recopying and rebuilding the entire source code, which was inefficient and time-consuming.

2) Advantages 
- Reduced image size and improved deployment speed: By removing unnecessary build tools and intermediate artifacts from the final image, the image size is reduced, leading to improvements in deployment speed.
- Easier dependency management: Managing dependencies only during the build stage, and including only essential elements in the final image, simplifies dependency management.
- Improved readability and maintainability: By separating the source code copying and building processes, the Dockerfile's structure becomes clearer, and maintenance becomes easier.

3) Disadvantages 
- Potential increase in total build process time: Performing the necessary tasks at each stage may take longer than a single-stage build. This can be somewhat mitigated with initial setup and cache optimization.
- Importance of cache management: Without the appropriate caching strategy for each build stage, unexpected increases in build time or build failures may occur.



### 프로젝트 구조
```
├── application // Web Server
├── batch
│    ├── etl  // Data ETL
├── common
│    ├──  data // ES Data
│    ├──  util // Common Utils
├── build.gradle.kts // root build file 
├── docker-compose-*.yml // docker file for Dev Env
├── Dockerfile // docker build file for Service 
├── settings.gradle.kts // setting for module 
```

## Previous Method
#### Dockerfile
```
FROM openjdk:17-jdk-slim

WORKDIR /app
COPY . .

RUN ./gradlew :application:build
ARG JAR_FILE=/app/application/build/libs/application-0.0.1-SNAPSHOT.jar
RUN cp ${JAR_FILE} app.jar

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```



## Improved Method

Build Stage Improvements:
- Optimized source code copying: Instead of COPY . ., selectively copying only the necessary files maximizes cache utilization and shortens build time.
- Optimized dependency download: Using the --no-daemon --refresh-dependencies option to accurately download only the needed dependencies, preventing unnecessary daemon process execution.

Execution Stage Improvements:
- Final image optimization: Copying only the executable JAR file generated in the build stage to the final image, incorporating only the minimal dependencies and files required for runtime. This strategy minimizes the size of the execution image and improves security and performance.


#### Dockerfile
```
# Build Stage
FROM gradle:7.6-jdk AS build
WORKDIR /app

# Copy necessary build files
COPY settings.gradle.kts .
COPY common/data/build.gradle.kts common/data/
COPY common/util/build.gradle.kts common/util/
COPY application/build.gradle.kts application/

# Download dependencies
RUN gradle --no-daemon --refresh-dependencies

# Copy source code
COPY common/data/src common/data/src
COPY common/util/src common/util/src
COPY application/src application/src

# Build application
RUN gradle :application:build -PskipTests --no-daemon

# Execution Stage
FROM openjdk:17-jdk-slim
WORKDIR /app

COPY --from=build /app/application/build/libs/application-*.jar app.jar

EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```