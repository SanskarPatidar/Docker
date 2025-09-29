```
WHY THIS FAILED:
The core, unavoidable issue is that for a multi-module Maven command like dependency:go-offline or
package to succeed, Maven must have access to the parent pom.xml AND all the sub-module folders (producer, consumer, etc.) at the same time to build its project "reactor".

The optimized caching approach of copying files piece by piece (COPY pom.xml, then COPY ${MODULE_DIR}/pom.xml, etc.)
fails because the generic Dockerfile doesn't know the names of all the other modules it needs to copy to satisfy Maven's reactor.

The only way to satisfy Maven's requirement and keep the Dockerfile generic is to use COPY . . in the builder stage.
This provides the complete project to Maven. While it seems less optimized for caching during the build, it is the only
robust way to guarantee the build succeeds, and it still results in the small, layered final image you want.
```
``` Dockerfile
# This Dockerfile accepts the module directory name as an argument
ARG MODULE_DIR
# Stage 1: The "Builder" - Compiles the application and creates the layered JAR
# This stage is optimized for caching by downloading dependencies before copying source code.
# Requires whole jdk for building the application, rather than just jre
FROM eclipse-temurin:17-jdk-alpine AS builder
RUN ls -R /

# setting up working directory from where the command will be run
WORKDIR /app

# 1. Copy the parent pom and the maven wrapper first for caching
# This layer will only be rebuilt if these files change.
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

# 2. Copy the pom.xml for the specific module we are building
# This layer is also cached separately.
COPY ${MODULE_DIR}/pom.xml ./${MODULE_DIR}/

# 3. Run dependency:go-offline. Maven will now download all the dependencies without building final jar
# defined in the parent and module poms. This is the slowest step and will
# now be cached, saving a huge amount of time on subsequent builds.
RUN ./mvnw dependency:go-offline

# 4. Copy the source code for the module. This is the layer that will
# change most often, and it's intentionally placed after the dependency download.
COPY ${MODULE_DIR}/src ./${MODULE_DIR}/

# 5. Build the entire project. This will be very fast because all
# dependencies are already downloaded and cached. It will just compile the code.
RUN ./mvnw package -DskipTests


# Stage 2: The "Extractor" - Extracts the layers from the JAR
FROM eclipse-temurin:17-jre-alpine AS extractor
# Just to keep things organized, create a directory for extraction
WORKDIR /extracted
# Copy the specific JAR from the builder stage
COPY --from=builder /app/${MODULE_DIR}/target/*.jar application.jar
# Extract the layers using Spring Boot's built-in support
RUN java -Djarmode=layertools -jar application.jar extract


# Stage 3: The "Final Image" - Assembles the lean, production image from the layers
FROM eclipse-temurin:17-jre-alpine
WORKDIR /application
# Copy the extracted layers from the extractor stage in the optimal order
COPY --from=extractor /extracted/dependencies/ ./
COPY --from=extractor /extracted/spring-boot-loader/ ./
COPY --from=extractor /extracted/snapshot-dependencies/ ./
COPY --from=extractor /extracted/application/ ./

# Application inside the container listens onn port 8080
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```
