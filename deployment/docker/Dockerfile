FROM wso2/wso2mi:4.2.0

# Switch to root to install dependencies
USER root

# Download and install Java Activation Framework
RUN wget -q https://repo1.maven.org/maven2/com/sun/activation/jakarta.activation/1.2.2/jakarta.activation-1.2.2.jar \
    -O ${WSO2_SERVER_HOME}/lib/jakarta.activation-1.2.2.jar

# Also add the older javax.activation for compatibility
RUN wget -q https://repo1.maven.org/maven2/javax/activation/activation/1.1.1/activation-1.1.1.jar \
    -O ${WSO2_SERVER_HOME}/lib/activation-1.1.1.jar

# Copy your application files
COPY target/bankingdocumentsystem_1.0.0.car ${WSO2_SERVER_HOME}/repository/deployment/server/carbonapps/
COPY deployment/docker/resources/ ${WSO2_SERVER_HOME}/repository/resources/security/
COPY deployment/deployment.toml ${WSO2_SERVER_HOME}/conf/

# Fix ownership of all files
RUN chown -R wso2carbon:wso2 ${WSO2_SERVER_HOME}

# Switch back to wso2carbon user
USER wso2carbon

EXPOSE 8290 8253

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8290/ || exit 1
