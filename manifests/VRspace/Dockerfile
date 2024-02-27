# Use Red Hat Universal Base Image (UBI)
FROM registry.access.redhat.com/ubi8/ubi

# Define build arguments
ARG openvidu_server_url
ARG openvidu_server_secret
ARG vrspace_server_url

# Install necessary packages
RUN dnf -y install maven git java-11-openjdk

# Set environment variables
ENV OPENVIDU_SERVER_URL=${openvidu_server_url}
ENV OPENVIDU_SERVER_SECRET=${openvidu_server_secret}
ENV VRSPACE_SERVER_URL=${vrspace_server_url}

# Set working directory
WORKDIR /app

# Clone the vrspace repository
RUN git clone ${VRSPACE_GIT_URL}

# Change working directory
WORKDIR /app/vrspace

# Build the application
RUN JAVA_HOME=/usr/lib/jvm/jre-11/ mvn clean install -DskipTests

# Expose port 8080
EXPOSE 8080

# Define the command to run the application
CMD ["/usr/lib/jvm/jre-11/bin/java", "-jar", "/app/vrspace/server/target/server-0.4.6-SNAPSHOT.jar"]
