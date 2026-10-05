# Based on the OpenSim Linux CI pipeline from:
# https://github.com/opensim-org/opensim-core/blob/be1c0548faacf3c9b1d911cd88e6e6ce864577f9/.github/workflows/continuous_integration.yml
FROM quay.io/pypa/manylinux_2_28_x86_64

# Install packages
# About package caching:
# https://protsenko.dev/infrastructure-security/purge-dnf-package-cache/
RUN dnf install -y \
    # Packages start
    gcc-c++ \
    make \
    cmake \
    libtool \
    autoconf \
    pkgconfig \
    gcc-gfortran \
    openblas-devel \
    lapack-devel \
    freeglut-devel \
    libXi-devel \
    libXmu-devel \
    doxygen \
    patchelf \
    zip \
    wget \
    pcre2-devel \
    git \
    python3-devel \
    # Packages end
    && \
    dnf clean all && \
    rm -rf /var/cache/dnf

# Setup uv
COPY --from=ghcr.io/astral-sh/uv:0.12.23 /uv /uvx /bin/
# Setup Python
RUN uv venv --python 3.11 /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# This version of numpy is required for AddBiomechanics
RUN uv pip install build numpy==1.24.4

RUN python --version
RUN python -c "import numpy; print(numpy.__version__); print(numpy.get_include())"

# Download OpenSim source (Oct 5, 2026 master branch)
RUN mkdir /opensim \
    && curl -SL https://github.com/opensim-org/opensim-core/archive/2aaa04b33463076edb06625693ad9bf98cf3bea4.tar.gz \
    | tar xzf - -C /opensim --strip-components=1

WORKDIR /opensim
RUN ls .
# Build dependencies
RUN cmake -S ./dependencies -B ./dependencies/build \
        -DCMAKE_INSTALL_PREFIX=./dependencies/install \
        --preset wheels-make-linux-release -LAH \
    && cmake --build ./dependencies/build \
        --preset wheels-make-linux-release \
        --parallel
# Configure and build opensim-core
RUN cmake . -B ./build \
  -DOPENSIM_DEPENDENCIES_DIR=./dependencies/install \
  -DCMAKE_INSTALL_PREFIX=~/opensim-core \
  -DPython3_EXECUTABLE=$(which python) \
  --preset wheels-make-linux-release -LAH \
  && cmake --build ./build \
    --preset wheels-make-linux-release \
    --parallel

# Install opensim-core
RUN cmake --build ./build --preset wheels-make-linux-release --target install
RUN (cd /root/opensim-core/sdk/Python && python -m build --wheel)

# Install the generated wheel in the environment
RUN uv pip install "$(find /root/opensim-core/sdk/Python/dist -maxdepth 1 -name '*.whl' -print -quit)"

ENV LD_LIBRARY_PATH="/opensim/dependencies/install/ezc3d/lib64:/root/opensim-core/sdk/lib"

# Install engine dependencies first (better caching)
COPY ./engine/requirements.txt /root/engine/requirements.txt
RUN uv pip install -r /root/engine/requirements.txt 

# Copy in the rest of the engine
COPY ./engine /root/engine

WORKDIR /root/engine
RUN uv pip install -e .

CMD ["/bin/bash"]