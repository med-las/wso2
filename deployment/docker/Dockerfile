# deployment/docker/Dockerfile
FROM --platform=linux/amd64 wso2/wso2mi:4.2.0

# Copy the CAR file
COPY target/bankingdocumentsystem_1.0.0.car ${WSO2_SERVER_HOME}/repository/deployment/server/carbonapps/

# Copy custom configurations
COPY deployment/docker/resources/ ${WSO2_SERVER_HOME}/repository/resources/security/

# Copy deployment configuration
COPY deployment/deployment.toml ${WSO2_SERVER_HOME}/conf/

# Set proper permissions
USER root
RUN chown -R wso2carbon:wso2 ${WSO2_SERVER_HOME}
USER wso2carbon

# Expose ports
EXPOSE 8290 8253

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8290/ || exit 1
