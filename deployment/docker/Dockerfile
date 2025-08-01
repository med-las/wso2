# Updated base image to 4.4.0
FROM wso2/wso2mi:4.4.0

# Copy the CAR file
COPY target/bankingdocumentsystem_1.0.0.car ${WSO2_SERVER_HOME}/repository/deployment/server/carbonapps/

# Copy custom configurations
COPY deployment/docker/resources/ ${WSO2_SERVER_HOME}/repository/resources/security/

# Copy deployment configuration  
COPY deployment/deployment.toml ${WSO2_SERVER_HOME}/conf/

# Set proper permissions
USER root
RUN chown -R wso2carbon:wso2 ${WSO2_SERVER_HOME}

# Optimized JVM settings for 4.4.0 with Java 17
ENV JAVA_OPTS="-Xms512m -Xmx1536m -XX:+UseG1GC -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -Djava.awt.headless=true -Dfile.encoding=UTF8"

USER wso2carbon

# Expose ports
EXPOSE 8290 8253

# Health check (4.4.0 has better health endpoints)
HEALTHCHECK --interval=30s --timeout=10s --start-period=90s --retries=5 \
  CMD curl -f http://localhost:8290/healthz || exit 1
