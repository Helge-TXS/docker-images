# LICENSE UPL 1.0
#
# Copyright (c) 2018, 2022 Oracle and/or its affiliates.
#
# ORACLE DOCKERFILES PROJECT
# --------------------------
# This is the Dockerfile for Oracle Database 19c
# 
# REQUIRED FILES TO BUILD THIS IMAGE
# ----------------------------------
# (1) db_home.zip
#     Download Oracle Database 19c Enterprise Edition or Standard Edition 2 for Linux x64
#     from http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html
#
# HOW TO BUILD THIS IMAGE
# -----------------------
# Put all downloaded files in the same directory as this Dockerfile
# Run: 
#      $ docker build -t oracle/database:19.3.0-${EDITION} . 
#
# Pull base image
# ---------------
ARG BASE_IMAGE=oraclelinux:9-slim
# hadolint ignore=DL3006
FROM ${BASE_IMAGE} AS base

# Labels
# ------
LABEL "provider"="Oracle"                                               \
      "issues"="https://github.com/oracle/docker-images/issues"         \
      "volume.data"="/opt/oracle/oradata"                               \
      "volume.setup.location1"="/opt/oracle/scripts/setup"              \
      "volume.setup.location2"="/docker-entrypoint-initdb.d/setup"      \
      "volume.startup.location1"="/opt/oracle/scripts/startup"          \
      "volume.startup.location2"="/docker-entrypoint-initdb.d/startup"  \
      "port.listener"="1521"                                            \
      "port.oemexpress"="5500"

# Argument to control removal of components not needed after db software installation
ARG SLIMMING=true
ARG INSTALL_FILE_1="LINUX.X64_193000_db_home.zip"
ARG OPATCH_UPDATE_FILE="p6880880_190000_Linux-x86-64.zip"
ARG RELEASE_UPDATE_FILE="p37260974_190000_Linux-x86-64.zip"

# Environment variables required for this build (do NOT change)
# -------------------------------------------------------------
ENV ORACLE_BASE=/opt/oracle \
    ORACLE_HOME=/opt/oracle/product/19c/dbhome_1 \
    INSTALL_DIR=/opt/install \
    INSTALL_FILE_1=$INSTALL_FILE_1 \
    OPATCH_UPDATE_FILE=$OPATCH_UPDATE_FILE \
    RELEASE_UPDATE_FILE=$RELEASE_UPDATE_FILE \
    INSTALL_RSP="db_inst.rsp" \
    CONFIG_RSP="dbca.rsp.tmpl" \
    PWD_FILE="setPassword.sh" \
    RUN_FILE="runOracle.sh" \
    START_FILE="startDB.sh" \
    CREATE_DB_FILE="createDB.sh" \
    CREATE_OBSERVER_FILE="createObserver.sh" \
    SETUP_LINUX_FILE="setupLinuxEnv.sh" \
    CHECK_SPACE_FILE="checkSpace.sh" \
    CHECK_DB_FILE="checkDBStatus.sh" \
    USER_SCRIPTS_FILE="runUserScripts.sh" \
    INSTALL_DB_BINARIES_FILE="installDBBinaries.sh" \
    RELINK_BINARY_FILE="relinkOracleBinary.sh" \
    CONFIG_TCPS_FILE="configTcps.sh" \
    SLIMMING=$SLIMMING \
    ENABLE_ARCHIVELOG=false \
    ARCHIVELOG_DIR_NAME=archive_logs \
    # To clone a database referred by PRIMARY_DB_CONN_STR
    CLONE_DB=false \
    # To create a standby database referred by PRIMARY_DB_CONN_STR
    STANDBY_DB=false \
    # Env var below should be in <HOST>:<PORT>/<SERVICE_NAME> format
    PRIMARY_DB_CONN_STR="" \
    # For creating dataguard observer
    DG_OBSERVER_ONLY=false \
    DG_OBSERVER_NAME="" \
    CHECKPOINT_FILE_EXTN=".created" \
    # Directory for keeping Oracle Wallet
    WALLET_DIR=""

# Use second ENV so that variable get substituted
ENV PATH=$ORACLE_HOME/bin:$ORACLE_HOME/OPatch/:/usr/sbin:$PATH \
    LD_LIBRARY_PATH=$ORACLE_HOME/lib:/usr/lib \
    CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib

# Copy files needed during both installation and runtime
# -------------
COPY $SETUP_LINUX_FILE $CHECK_SPACE_FILE $INSTALL_DIR/
COPY $RUN_FILE $START_FILE $CREATE_DB_FILE $CREATE_OBSERVER_FILE $CONFIG_RSP $PWD_FILE $CHECK_DB_FILE $USER_SCRIPTS_FILE $RELINK_BINARY_FILE $CONFIG_TCPS_FILE $ORACLE_BASE/

RUN chmod ug+x $INSTALL_DIR/*.sh && \
    sync && \
    $INSTALL_DIR/$CHECK_SPACE_FILE && \
    $INSTALL_DIR/$SETUP_LINUX_FILE && \
    rm -rf $INSTALL_DIR



#############################################
# -------------------------------------------
# Start new stage for installing the database
# -------------------------------------------
#############################################

FROM base AS builder

ARG DB_EDITION

# Copy DB install file
COPY --chown=oracle:dba $INSTALL_FILE_1 $OPATCH_UPDATE_FILE $RELEASE_UPDATE_FILE $INSTALL_RSP $INSTALL_DB_BINARIES_FILE $INSTALL_DIR/
#RUN chown oracle:dba $INSTALL_DIR

# Install DB software binaries
USER oracle
RUN chmod ug+x "$INSTALL_DIR"/*.sh && \
    sync && \
    "$INSTALL_DIR"/"$INSTALL_DB_BINARIES_FILE" $DB_EDITION && \
    rm -rf $ORACLE_HOME/.patch_storage $ORACLE_HOME/.opatchauto_storage



#############################################
# -------------------------------------------
# Start new layer for database runtime
# -------------------------------------------
#############################################

FROM base

USER oracle
COPY --chown=oracle:dba --from=builder $ORACLE_BASE $ORACLE_BASE

USER root
RUN "$ORACLE_BASE"/oraInventory/orainstRoot.sh && \
    "$ORACLE_HOME"/root.sh

USER oracle
WORKDIR /home/oracle

# Add a bashrc file to capitalize ORACLE_SID in the environment
# hadolint ignore=SC2016
RUN echo 'ORACLE_SID=${ORACLE_SID:-ORCLCDB}; export ORACLE_SID=${ORACLE_SID^^}' > .bashrc

HEALTHCHECK --interval=1m --start-period=5m --timeout=30s \
   CMD "$ORACLE_BASE/$CHECK_DB_FILE" >/dev/null || exit 1

# Define default command to start Oracle Database. 
CMD [ "/bin/bash", "-c", "exec $ORACLE_BASE/$RUN_FILE" ]
