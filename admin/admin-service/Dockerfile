# syntax=docker/dockerfile:1
############################
# 1) Build stage (Maven + JDK 11)
############################
FROM maven:3.9-eclipse-temurin-11 AS build
WORKDIR /workspace
# Copy pom first to leverage dependency cache
COPY pom.xml .
RUN mvn -q -B -DskipTests dependency:go-offline
# Copy sources and build
COPY src ./src
RUN mvn -q -B -DskipTests package
############################
# 2) Runtime stage (JRE 11)
############################
FROM eclipse-temurin:11-jre
# ----- Build-time metadata (labels) -----
ARG SOURCE
ARG COMMIT_HASH
ARG COMMIT_ID
ARG BUILD_TIME
LABEL source="${SOURCE}" \
      commit_hash="${COMMIT_HASH}" \
      commit_id="${COMMIT_ID}" \
      build_time="${BUILD_TIME}"
# ----- Build-time args for config (exposed as ENV for runtime overrides) -----
ARG spring_config_label
ARG active_profile
ARG spring_config_url
ARG artifactory_url
ARG iam_adapter_url
ARG is_glowroot
ENV spring_config_label_env="${spring_config_label}" \
    active_profile_env="${active_profile}" \
    spring_config_url_env="${spring_config_url}" \
    artifactory_url_env="${artifactory_url}" \
    iam_adapter_url_env="${iam_adapter_url}" \
    is_glowroot_env="${is_glowroot}"
# ----- Create non-root user & install minimal tools (wget/unzip for downloads) -----
ARG container_user=mosip
ARG container_user_group=mosip
ARG container_user_uid=1001
ARG container_user_gid=1001
RUN apt-get update \
 && apt-get install -y --no-install-recommends wget unzip ca-certificates \
 && groupadd -g "${container_user_gid}" "${container_user_group}" \
 && useradd -u "${container_user_uid}" -g "${container_user_group}" -s /bin/sh -m "${container_user}" \
 && rm -rf /var/lib/apt/lists/*
WORKDIR /home/${container_user}
ENV work_dir=/home/${container_user}
# Additional jars folder (Spring Boot loader.path)
ARG loader_path=${work_dir}/additional_jars/
RUN mkdir -p "${loader_path}"
ENV loader_path_env="${loader_path}"
# Copy built JAR from the build stage
# Adjust the pattern if your artifact name differs
COPY --from=build /workspace/target/*.jar /home/${container_user}/admin-service.jar
# Permissions & drop privileges
RUN chown -R ${container_user}:${container_user} /home/${container_user}
USER ${container_user_uid}:${container_user_gid}
# App port (change if your app uses another)
EXPOSE 8098
# Optional healthcheck (tweak path/port as needed)
# HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=5 \
#   CMD wget -qO- http://127.0.0.1:8098/actuator/health | grep -q '"status":"UP"' || exit 1
# Start command:
# - If glowroot requested (is_glowroot_env=present), download & enable agent.
# - Always try to fetch IAM adapter JAR into loader path if URL provided.
# NOTE: JVM -D options must precede -jar.
CMD if [ "$is_glowroot_env" = "present" ]; then \
      echo "Glowroot enabled. Downloading..."; \
      wget -q "${artifactory_url_env}/artifactory/libs-release-local/io/mosip/testing/glowroot.zip" -O glowroot.zip && \
      unzip -q glowroot.zip && rm -f glowroot.zip && \
      sed -i 's/<service_name>/kernel-auditmanager-service/g' glowroot/glowroot.properties ; \
    fi ; \
    if [ -n "${iam_adapter_url_env}" ]; then \
      echo "Fetching IAM adapter jar..."; \
      mkdir -p "${loader_path_env}" && \
      wget -q "${iam_adapter_url_env}" -O "${loader_path_env}/kernel-auth-adapter.jar" || echo "WARN: IAM adapter download failed"; \
    else \
      echo "WARN: iam_adapter_url_env empty; skipping IAM adapter download."; \
    fi ; \
    if [ "$is_glowroot_env" = "present" ]; then \
      exec java -javaagent:glowroot/glowroot.jar \
        -Dloader.path="${loader_path_env}" \
        -Dspring.cloud.config.label="${spring_config_label_env}" \
        -Dspring.profiles.active="${active_profile_env}" \
        -Dspring.cloud.config.uri="${spring_config_url_env}" \
        -jar admin-service.jar ; \
    else \
      exec java \
        -Dloader.path="${loader_path_env}" \
        -Dspring.cloud.config.label="${spring_config_label_env}" \
        -Dspring.profiles.active="${active_profile_env}" \
        -Dspring.cloud.config.uri="${spring_config_url_env}" \
        -jar admin-service.jar ; \
    fiFROM openjdk:11

ARG SOURCE
ARG COMMIT_HASH
ARG COMMIT_ID
ARG BUILD_TIME
LABEL source=${SOURCE}
LABEL commit_hash=${COMMIT_HASH}
LABEL commit_id=${COMMIT_ID}
LABEL build_time=${BUILD_TIME}

# can be passed during Docker build as build time environment for github branch to pickup configuration from.
ARG spring_config_label

# can be passed during Docker build as build time environment for spring profiles active 
ARG active_profile

# can be passed during Docker build as build time environment for config server URL 
ARG spring_config_url

# can be passed during Docker build as build time environment for artifactory URL
ARG artifactory_url

# can be passed during Docker build as build time environment for glowroot
ARG is_glowroot

# environment variable to pass active profile such as DEV, QA etc at docker runtime
ENV active_profile_env=${active_profile}

# environment variable to pass github branch to pickup configuration from, at docker runtime
ENV spring_config_label_env=${spring_config_label}

# environment variable to pass spring configuration url, at docker runtime
ENV spring_config_url_env=${spring_config_url}

ENV is_glowroot_env=${is_glowroot}

# environment variable to pass artifactory url, at docker runtime
ENV artifactory_url_env=${artifactory_url}

# environment variable to pass iam_adapter url, at docker runtime
ENV iam_adapter_url_env=${iam_adapter_url}

# can be passed during Docker build as build time environment for github branch to pickup configuration from.
ARG container_user=mosip

# can be passed during Docker build as build time environment for github branch to pickup configuration from.
ARG container_user_group=mosip

# can be passed during Docker build as build time environment for github branch to pickup configuration from.
ARG container_user_uid=1001

# can be passed during Docker build as build time environment for github branch to pickup configuration from.
ARG container_user_gid=1001

# install packages and create user
RUN apt-get -y update \
&& apt-get install -y unzip \
&& groupadd -g ${container_user_gid} ${container_user_group} \
&& useradd -u ${container_user_uid} -g ${container_user_group} -s /bin/sh -m ${container_user}

# set working directory for the user
WORKDIR /home/${container_user}

ENV work_dir=/home/${container_user}

ARG loader_path=${work_dir}/additional_jars/

RUN mkdir -p ${loader_path}

ENV loader_path_env=${loader_path}

COPY ./target/admin-service-*.jar admin-service.jar

# change permissions of file inside working dir
RUN chown -R ${container_user}:${container_user} /home/${container_user}

# select container user for all tasks
USER ${container_user_uid}:${container_user_gid}

EXPOSE 8098

CMD if [ "$is_glowroot_env" = "present" ]; then \
    wget -q --show-progress "${artifactory_url_env}"/artifactory/libs-release-local/io/mosip/testing/glowroot.zip ; \
    unzip glowroot.zip ; \
    rm -rf glowroot.zip ; \
    sed -i 's/<service_name>/kernel-auditmanager-service/g' glowroot/glowroot.properties ; \
    wget -q --show-progress "${iam_adapter_url_env}" -O "${loader_path_env}"/kernel-auth-adapter.jar; \
    java -jar  -javaagent:glowroot/glowroot.jar -Dloader.path="${loader_path_env}" -jar -Dspring.cloud.config.label="${spring_config_label_env}" -Dspring.profiles.active="${active_profile_env}"  -Dspring.cloud.config.uri="${spring_config_url_env}" admin-service.jar; \
    else \
    wget -q --show-progress "${iam_adapter_url_env}" -O "${loader_path_env}"/kernel-auth-adapter.jar; \
    java -jar -Dloader.path="${loader_path_env}" -Dspring.cloud.config.label="${spring_config_label_env}" -Dspring.profiles.active="${active_profile_env}" -Dspring.cloud.config.uri="${spring_config_url_env}" admin-service.jar; \
    fi
