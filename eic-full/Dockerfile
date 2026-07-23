# Full EIC software stack (adds EPIC geometry and EICrecon on top of eic-base)
#
# Build:
#   docker build -t eicdev/eic-full:latest .
#   docker build -t eicdev/eic-full:latest --build-arg BUILD_THREADS=24 .
#   docker buildx build --tag eicdev/eic-full:latest --push .
#   docker buildx build --tag eicdev/eic-full:latest --no-cache --push .
#
# Run:
#   docker run --rm -it --init eicdev/eic-full:latest
#
# Build with buildx debug:
#   BUILDX_EXPERIMENTAL=1 docker buildx debug --invoke bash build --progress=plain
#   In PowerShell:  $env:BUILDX_EXPERIMENTAL="1"

FROM eicdev/eic-base:latest

WORKDIR ${APP_ROOT}

# ---------------------------------------------------------------------------#
#  Build-time arguments & environment                                        #
# ---------------------------------------------------------------------------#
ARG BUILD_THREADS=8
ARG CXX_STANDARD=20


### ---------------------------------------------------------------------- ###
###          EPIC build (ePIC detector geometry)                            ###
### ---------------------------------------------------------------------- ###
ARG VERSION_EPIC="main"
ENV EPIC_SRC_PATH=${APP_ROOT}/epic-src \
    EPIC_BLD_PATH=${APP_ROOT}/epic-build \
    EPIC_INS_PATH=${APP_ROOT}/epic

RUN set -ex && \
    git clone --branch ${VERSION_EPIC} --depth 1 https://github.com/eic/epic.git "${EPIC_SRC_PATH}" && \
    mkdir -p "${EPIC_BLD_PATH}" && cd "${EPIC_BLD_PATH}" && \
    cmake \
      -DCMAKE_INSTALL_PREFIX="${EPIC_INS_PATH}" \
      -DCMAKE_BUILD_TYPE=RelWithDebInfo \
      -DCMAKE_CXX_STANDARD=${CXX_STANDARD} \
      "${EPIC_SRC_PATH}" && \
    cmake --build . --target install -j"${BUILD_THREADS}" && \
    rm -rf "${EPIC_BLD_PATH}" "${EPIC_SRC_PATH}"

ENV DETECTOR_PATH="${EPIC_INS_PATH}/share/epic" \
    BEAMLINE=epic \
    BEAMLINE_PATH="${EPIC_INS_PATH}/share/epic" \
    BEAMLINE_CONFIG=epic \
    PATH="${EPIC_INS_PATH}/bin:${PATH}" \
    LD_LIBRARY_PATH="${EPIC_INS_PATH}/lib:${LD_LIBRARY_PATH}" \
    CMAKE_PREFIX_PATH="${EPIC_INS_PATH}:${CMAKE_PREFIX_PATH}" \
    DETECTOR="epic" \
    DETECTOR_CONFIG="epic" \
    DETECTOR_VERSION="v0.0.0-unknown"


### ---------------------------------------------------------------------- ###
###          EICrecon build (EIC Reconstruction Framework)                  ###
### ---------------------------------------------------------------------- ###
ARG VERSION_EICRECON=main
ENV EICRECON_SRC_PATH=${APP_ROOT}/eicrecon-src \
    EICRECON_BLD_PATH=${APP_ROOT}/eicrecon-build \
    EICRECON_INS_PATH=${APP_ROOT}/eicrecon

RUN set -ex && \
    git clone --branch ${VERSION_EICRECON} --depth 1 https://github.com/eic/eicrecon.git "${EICRECON_SRC_PATH}" && \
    mkdir -p "${EICRECON_BLD_PATH}" && cd "${EICRECON_BLD_PATH}" && \
    cmake \
      -DCMAKE_INSTALL_PREFIX="${EICRECON_INS_PATH}" \
      -DCMAKE_BUILD_TYPE=RelWithDebInfo \
      -DCMAKE_CXX_STANDARD=${CXX_STANDARD} \
      -DCMAKE_CXX_FLAGS="-isystem ${ALGORITHMS_INS_PATH}/include -isystem ${SPDLOG_INS_PATH}/include -isystem ${DD4HEP_INS_PATH}/include" \
      "${EICRECON_SRC_PATH}" && \
    cmake --build . --target install -j"${BUILD_THREADS}" && \
    rm -rf "${EICRECON_BLD_PATH}" "${EICRECON_SRC_PATH}"

ENV eicrecon_HOME=${EICRECON_INS_PATH} \
    JANA_PLUGIN_PATH="${EICRECON_INS_PATH}/lib/EICrecon/plugins:${JANA_PLUGIN_PATH}" \
    PATH="${EICRECON_INS_PATH}/bin:${PATH}" \
    LD_LIBRARY_PATH="${EICRECON_INS_PATH}/lib:${LD_LIBRARY_PATH}" \
    CMAKE_PREFIX_PATH="${EICRECON_INS_PATH}:${CMAKE_PREFIX_PATH}"


### ---------------------------------------------------------------------- ###
###          ilreco + g4cal (emcal PbWO4 sim/reco, LL-profile production)   ###
### ---------------------------------------------------------------------- ###
# Geant4 comes from eic-base (${GEANT4_INS_PATH}); jobs must
# `source $GEANT4_INS_PATH/bin/geant4.sh` for the dataset env vars.
ARG VERSION_ILRECO=master
ENV ILRECO_SRC_PATH=${APP_ROOT}/ilreco-src \
    ILRECO_BLD_PATH=${APP_ROOT}/ilreco-build \
    ILRECO_INS_PATH=${APP_ROOT}/ilreco

RUN set -ex && \
    git clone --branch ${VERSION_ILRECO} --depth 1 https://github.com/emcal/ilreco.git "${ILRECO_SRC_PATH}" && \
    mkdir -p "${ILRECO_BLD_PATH}" && cd "${ILRECO_BLD_PATH}" && \
    cmake \
      -DCMAKE_INSTALL_PREFIX="${ILRECO_INS_PATH}" \
      -DCMAKE_BUILD_TYPE=Release \
      -DILRECO_BUILD_TESTS=OFF \
      "${ILRECO_SRC_PATH}" && \
    cmake --build . --target install -j"${BUILD_THREADS}" && \
    rm -rf "${ILRECO_BLD_PATH}" "${ILRECO_SRC_PATH}"

ENV LD_LIBRARY_PATH="${ILRECO_INS_PATH}/lib:${LD_LIBRARY_PATH}" \
    CMAKE_PREFIX_PATH="${ILRECO_INS_PATH}:${CMAKE_PREFIX_PATH}"

# g4cal: calsim/calreco binaries (finds the ilreco install above) plus the
# g4cal python package (python3 -m g4cal.llprof fills profile histograms
# in-job so farm jobs ship compact npz partials instead of raw hits).
ARG VERSION_G4CAL=main
ENV G4CAL_SRC_PATH=${APP_ROOT}/g4cal-src \
    G4CAL_BLD_PATH=${APP_ROOT}/g4cal-build \
    G4CAL_INS_PATH=${APP_ROOT}/g4cal

RUN set -ex && \
    git clone --branch ${VERSION_G4CAL} --depth 1 https://github.com/emcal/g4cal.git "${G4CAL_SRC_PATH}" && \
    mkdir -p "${G4CAL_BLD_PATH}" && cd "${G4CAL_BLD_PATH}" && \
    cmake \
      -DCMAKE_INSTALL_PREFIX="${G4CAL_INS_PATH}" \
      -DCMAKE_BUILD_TYPE=Release \
      "${G4CAL_SRC_PATH}" && \
    cmake --build . --target install -j"${BUILD_THREADS}" && \
    pip3 install --no-cache-dir --break-system-packages "${G4CAL_SRC_PATH}" && \
    rm -rf "${G4CAL_BLD_PATH}" "${G4CAL_SRC_PATH}"

ENV PATH="${G4CAL_INS_PATH}/bin:${PATH}"


SHELL ["/bin/bash", "-c"]
