FROM tomeichlersmith/hps-env:v3.3.0
MAINTAINER Tom Eichlersmith <eichl008@umn.edu>

# need to choose and install a version of hps-java
#   not sure what the best way to define install destination
#   for a java project is
RUN mkdir ${__prefix}/hps &&\
    git clone https://github.com/JeffersonLab/hps-java ${__prefix}/hps/java &&\
    cd ${__prefix}/hps/java &&\
    mvn \
      -DskipTests \
      -Dcheckstyle.skip \
    &&\
    cp -t ${__prefix}/hps/java/ \
      ~/.m2/repository/org/lcsim/lcio/2.7.4-SNAPSHOT/lcio-2.7.4-SNAPSHOT-bin.jar
ENV HPS_JAVA_BIN_JAR=${__prefix}/hps/java/distribution/target/hps-distribution-5.2-SNAPSHOT-bin.jar

# download and unpack fieldmaps
#   destination is chosen to be what is expected from hps-mc defaults
RUN git clone https://github.com/JeffersonLab/hps-fieldmaps ${__prefix}/share/fieldmap &&\
    cd ${__prefix}/share/fieldmap &&\
    ./unzip.sh

# install hps-mc for its recipes
#   use its CMake to copy in MadGraph
#   as well as build stdhep utility programs
RUN git clone https://github.com/JeffersonLab/hps-mc src &&\
    cmake \
      -DHPSMC_ENABLE_STDHEP=ON \
      -DHPSMC_ENABLE_MADGRAPH=ON \
      -DCMAKE_INSTALL_PREFIX=${__prefix} \
      -B src/build \
      -S src &&\
    cmake --build src/build --target install &&\
    rm -rf src
ENV PYTHONPATH=${PYTHONPATH}:${__prefix}/lib/python
ENV HPSMC_DIR=${__prefix}

# install hpstr for tuplization
RUN git clone https://github.com/tomeichlersmith/hpstr src \
        --branch v0.1.1 \
        --depth 1 &&\
    cmake \
        -DCMAKE_INSTALL_PREFIX=${__prefix} \
        -B src/build \
        -S src &&\
    cmake --build src/build --target install &&\
    rm -rf src

# copy in a hps-mc config file that users can supply when
# running hps-mc inside this container
COPY ./container.cfg /usr/local/share/
