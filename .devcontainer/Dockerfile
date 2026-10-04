FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# ------------------------------------------------------------
# Android environment
# ------------------------------------------------------------

ENV ANDROID_HOME=/opt/android-sdk
ENV ANDROID_SDK_ROOT=/opt/android-sdk

ENV PATH="${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools:${ANDROID_HOME}/emulator:${PATH}"

# ------------------------------------------------------------
# Basic packages + JDK
# ------------------------------------------------------------

RUN apt-get update && apt-get install -y \
    ca-certificates \
    curl \
    wget \
    unzip \
    git \
    git-lfs \
    openssh-client \
    sudo \
    jq \
    ripgrep \
    fd-find \
    openjdk-17-jdk \
    && rm -rf /var/lib/apt/lists/*

# ------------------------------------------------------------
# Android command-line tools
# ------------------------------------------------------------

RUN mkdir -p ${ANDROID_HOME}/cmdline-tools \
    && curl -fsSL \
       https://dl.google.com/android/repository/commandlinetools-linux-15859902_latest.zip \
       -o /tmp/android-tools.zip \
    && unzip -q /tmp/android-tools.zip \
       -d ${ANDROID_HOME}/cmdline-tools \
    && mv ${ANDROID_HOME}/cmdline-tools/cmdline-tools \
       ${ANDROID_HOME}/cmdline-tools/latest \
    && rm /tmp/android-tools.zip

# ------------------------------------------------------------
# Android SDK packages
# ------------------------------------------------------------

RUN yes | sdkmanager --licenses >/dev/null || true

RUN sdkmanager \
    "platform-tools" \
    "platforms;android-36" \
    "build-tools;36.0.0"

# ------------------------------------------------------------
# Developer user
# ------------------------------------------------------------

# Ubuntu 24.04 provides the "ubuntu" user.
# Give it ownership of the Android SDK so SDK tools can be
# used without root permissions.

RUN chown -R ubuntu:ubuntu ${ANDROID_HOME}

USER ubuntu

WORKDIR /workspace
