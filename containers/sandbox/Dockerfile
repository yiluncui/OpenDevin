FROM public.ecr.aws/ubuntu/ubuntu:22.04 AS core

# install basic packages
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    git \
    vim \
    nano \
    unzip \
    zip \
    python3 \
    python3-pip \
    python3-venv \
    python3-dev \
    build-essential \
    openssh-server \
    sudo \
    bash \
    gcc \
    jq \
    g++ \
    make \
    iproute2 \
    libgl1-mesa-glx \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p -m0755 /var/run/sshd

# symlink python3 to python
RUN ln -s /usr/bin/python3 /usr/bin/python

# install basic dependencies for CodeActAgent
RUN pip3 install --upgrade pip
RUN pip3 install jupyterlab notebook jupyter_kernel_gateway flake8
# TODO: those dependencies are needed for agentskills, we should pack them in a new sandbox image
RUN pip3 install python-docx PyPDF2 python-pptx pylatexenc openai opencv-python


# Install AWS CLI v2
# https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html
RUN curl https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip -o /tmp/awscliv2.zip \
    && unzip -q /tmp/awscliv2.zip -d /opt \
    && /opt/aws/install --update -i /usr/local/aws-cli -b /usr/local/bin \
    && rm /tmp/awscliv2.zip \
    && rm -rf /opt/aws \
    && aws --version

#****************        DOCKER    *********************************************
ARG DOCKER_BUCKET="download.docker.com"
ARG DOCKER_CHANNEL="stable"
ARG DIND_COMMIT="3b5fac462d21ca164b3778647420016315289034"
ARG DOCKER_COMPOSE_VERSION="2.23.1"
ARG DOCKER_BUILDX_VERSION="0.11.0"
ARG SRC_DIR="/usr/src"

ARG DOCKER_SHA256="544262F4A3621222AFB79960BFAD4D486935DAB80893478B5CC9CF8EBAF409AE"
ARG DOCKER_VERSION="23.0.6"

# Install Docker
RUN set -ex \
    && curl -fSL "https://${DOCKER_BUCKET}/linux/static/${DOCKER_CHANNEL}/x86_64/docker-${DOCKER_VERSION}.tgz" -o docker.tgz \
    && echo "${DOCKER_SHA256} *docker.tgz" | sha256sum -c - \
    && tar --extract --file docker.tgz --strip-components 1  --directory /usr/local/bin/ \
    && rm docker.tgz \
    && docker -v \
    # replace runc package to resolve CVE-2024-21626
    && apt-get update && apt-get -y install runc \
    && rm -f /usr/local/bin/runc \
    && ln -s /usr/sbin/runc /usr/local/bin/runc \
    && runc -v \
    # set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
    && addgroup dockremap \
    && useradd -g dockremap dockremap \
    && echo 'dockremap:165536:65536' >> /etc/subuid \
    && echo 'dockremap:165536:65536' >> /etc/subgid \
    && wget -q "https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind" -O /usr/local/bin/dind \
    # Install docker compose as docker plugin and maintain docker-compose usage
    && mkdir -p /usr/local/lib/docker/cli-plugins \
    && curl -L https://github.com/docker/compose/releases/download/v${DOCKER_COMPOSE_VERSION}/docker-compose-Linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose \
    && chmod +x /usr/local/bin/dind /usr/local/lib/docker/cli-plugins/docker-compose \
    && ln -s /usr/local/lib/docker/cli-plugins/docker-compose /usr/local/bin/docker-compose \
    # Ensure docker-compose and docker compose work
    && docker-compose version \
    && docker compose version \
    # Add docker buildx tool
    && curl -L https://github.com/docker/buildx/releases/download/v${DOCKER_BUILDX_VERSION}/buildx-v${DOCKER_BUILDX_VERSION}.linux-amd64 -o /usr/local/lib/docker/cli-plugins/docker-buildx \
    && chmod +x /usr/local/lib/docker/cli-plugins/docker-buildx \
    && ln -s /usr/local/lib/docker/cli-plugins/docker-buildx /usr/local/bin/docker-buildx \
    # Ensure docker-buildx works
    && docker-buildx version \
    && docker buildx version

VOLUME /var/lib/docker
#*********************** END  DOCKER  ****************************


#****************      JAVA     ****************************************************
ENV JAVA_17_HOME="/usr/lib/jvm/java-17-amazon-corretto" \
    JDK_17_HOME="/usr/lib/jvm/java-17-amazon-corretto" \
    JRE_17_HOME="/usr/lib/jvm/java-17-amazon-corretto" \
    JAVA_21_HOME="/usr/lib/jvm/java-21-amazon-corretto" \
    JDK_21_HOME="/usr/lib/jvm/java-21-amazon-corretto" \
    JRE_21_HOME="/usr/lib/jvm/java-21-amazon-corretto" \
    JAVA_11_HOME="/usr/lib/jvm/java-11-amazon-corretto" \
    JDK_11_HOME="/usr/lib/jvm/java-11-amazon-corretto" \
    JRE_11_HOME="/usr/lib/jvm/java-11-amazon-corretto" \
    JAVA_8_HOME="/usr/lib/jvm/java-1.8.0-amazon-corretto" \
    JDK_8_HOME="/usr/lib/jvm/java-1.8.0-amazon-corretto" \
    JRE_8_HOME="/usr/lib/jvm/java-1.8.0-amazon-corretto"
ARG ANT_VERSION=1.10.14
ARG MAVEN_HOME="/opt/maven"
ARG MAVEN_VERSION=3.9.5
ARG GRADLE_VERSION=8.3
ARG SBT_VERSION=1.9.6
ARG GRADLE_PATH="$SRC_DIR/gradle"
ARG ANT_DOWNLOAD_SHA512="4e74b382dd8271f9eac9fef69ba94751fb8a8356dbd995c4d642f2dad33de77bd37d4001d6c8f4f0ef6789529754968f0c1b6376668033c8904c6ec84543332a"
ARG MAVEN_DOWNLOAD_SHA512="4810523ba025104106567d8a15a8aa19db35068c8c8be19e30b219a1d7e83bcab96124bf86dc424b1cd3c5edba25d69ec0b31751c136f88975d15406cab3842b"
ARG GRADLE_DOWNLOADS_SHA256="bb09982fdf52718e4c7b25023d10df6d35a5fff969860bdf5a5bd27a3ab27a9e 8.3"
ARG SBT_DOWNLOAD_SHA256="923d7917ccb99a9fd985f4abfd81caacaed42284e67d3f7696cc5239e7c595cb"

ARG MAVEN_CONFIG_HOME="/root/.m2"

ENV JAVA_HOME="$JAVA_17_HOME" \
    JDK_HOME="$JDK_17_HOME" \
    JRE_HOME="$JRE_17_HOME"

ENV PATH="${PATH}:/opt/tools"

RUN set -ex \
    && apt-get update \
    && apt-get install -y -qq software-properties-common apt-utils sudo \
    # Install Corretto 17
    && wget -O - https://apt.corretto.aws/corretto.key | sudo gpg --dearmor -o /usr/share/keyrings/corretto-keyring.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/corretto-keyring.gpg] https://apt.corretto.aws stable main" | sudo tee /etc/apt/sources.list.d/corretto.list \
    && apt-get update \
    && apt-get install -y -qq java-17-amazon-corretto-jdk \
    && apt-get install -y -qq java-21-amazon-corretto-jdk \
    && apt-get install -y -qq java-11-amazon-corretto-jdk \
    && apt-get install -y -qq java-1.8.0-amazon-corretto-jdk \
    && apt-get install -y -qq --no-install-recommends ca-certificates-java \
    # Ensure Java cacerts symlink points to valid location
    && update-ca-certificates -f \
    && apt-get update \
    && for tool_path in $JAVA_HOME/bin/*; do \
          tool=`basename $tool_path`; \
          update-alternatives --install /usr/bin/$tool $tool $tool_path 10000; \
          update-alternatives --set $tool $tool_path; \
        done \
     && rm $JAVA_HOME/lib/security/cacerts && ln -s /etc/ssl/certs/java/cacerts $JAVA_HOME/lib/security/cacerts \
    # Install Ant
    && curl -LSso /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz https://archive.apache.org/dist/ant/binaries/apache-ant-$ANT_VERSION-bin.tar.gz  \
    && echo "$ANT_DOWNLOAD_SHA512 /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz" | sha512sum -c - \
    && tar -xzf /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz -C /opt \
    && rm /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz \
    && update-alternatives --install /usr/bin/ant ant /opt/apache-ant-$ANT_VERSION/bin/ant 10000

RUN set -ex \
    # Install Maven
    && mkdir -p $MAVEN_HOME \
    && curl -LSso /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz https://archive.apache.org/dist/maven/maven-3/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz \
    && echo "$MAVEN_DOWNLOAD_SHA512 /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz" | sha512sum -c - \
    && tar xzf /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz -C $MAVEN_HOME --strip-components=1 \
    && rm /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz \
    && update-alternatives --install /usr/bin/mvn mvn /opt/maven/bin/mvn 10000 \
    && mkdir -p $MAVEN_CONFIG_HOME \
    # Install Gradle
    && mkdir -p $GRADLE_PATH \
    && wget -q "https://services.gradle.org/distributions/gradle-$GRADLE_VERSION-all.zip" -O "$GRADLE_PATH/gradle-$GRADLE_VERSION-all.zip" \
    && unzip -q "$GRADLE_PATH/gradle-$GRADLE_VERSION-all.zip" -d /usr/local \
    && echo "$GRADLE_DOWNLOADS_SHA256" | grep "$GRADLE_VERSION" | sed "s|$GRADLE_VERSION|$GRADLE_PATH/gradle-$GRADLE_VERSION-all.zip|" | sha256sum -c - \
    && rm "$GRADLE_PATH/gradle-$GRADLE_VERSION-all.zip" \
    && mkdir "/tmp/gradle-$GRADLE_VERSION" \
    && "/usr/local/gradle-$GRADLE_VERSION/bin/gradle" -p "/tmp/gradle-$GRADLE_VERSION" init \
    && "/usr/local/gradle-$GRADLE_VERSION/bin/gradle" -p "/tmp/gradle-$GRADLE_VERSION" wrapper \
    # Android Studio uses the "-all" distribution for it's wrapper script.
    && perl -pi -e "s/gradle-$GRADLE_VERSION-bin.zip/gradle-$GRADLE_VERSION-all.zip/" "/tmp/gradle-$GRADLE_VERSION/gradle/wrapper/gradle-wrapper.properties" \
    && "/tmp/gradle-$GRADLE_VERSION/gradlew" -p "/tmp/gradle-$GRADLE_VERSION" init \
    && rm -rf "/tmp/gradle-$GRADLE_VERSION"  \
    # Install default GRADLE_VERSION to path
    && ln -s /usr/local/gradle-$GRADLE_VERSION/bin/gradle /usr/bin/gradle \
    && rm -rf $GRADLE_PATH \
    # Install SBT
    && curl -fSL "https://github.com/sbt/sbt/releases/download/v${SBT_VERSION}/sbt-${SBT_VERSION}.tgz" -o sbt.tgz \
    && echo "${SBT_DOWNLOAD_SHA256} *sbt.tgz" | sha256sum -c - \
    && tar xzf sbt.tgz -C /usr/local/bin/ \
    && rm sbt.tgz

ENV PATH "/usr/local/bin/sbt/bin:$PATH"
RUN sbt version -Dsbt.rootdir=true
# Cleanup
RUN rm -fr /var/lib/apt/lists/* /tmp/* /var/tmp/* \
    && apt-get clean
#****************     END JAVA     ****************************************************