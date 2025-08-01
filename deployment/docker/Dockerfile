FROM wso2/wso2mi:4.2.0

# Copy the CAR file
COPY target/bankingdocumentsystem_1.0.0.car ${WSO2_SERVER_HOME}/repository/deployment/server/carbonapps/

# Copy custom configurations
COPY deployment/docker/resources/ ${WSO2_SERVER_HOME}/repository/resources/security/

# Copy deployment configuration
COPY deployment/deployment.toml ${WSO2_SERVER_HOME}/conf/

# Set proper permissions
USER root
RUN chown -R wso2carbon:wso2 ${WSO2_SERVER_HOME}

# Fixed JVM options - removed deprecated UseCGroupMemoryLimitForHeap
ENV JAVA_OPTS="-Xms512m -Xmx1024m -XX:+UseG1GC -Djava.awt.headless=true -Dfile.encoding=UTF8 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/"

USER wso2carbon

# Expose ports
EXPOSE 8290 8253

# Health check
HEALTHCHECK --interval=45s --timeout=10s --start-period=120s --retries=5 \
  CMD curl -f http://localhost:8290/services || exit 1
