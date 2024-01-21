# Copyright 2021 DeepMind Technologies Limited
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

ARG CUDA=11.7.1
FROM nvidia/cuda:${CUDA}-cudnn8-runtime-ubuntu18.04
# FROM directive resets ARGS, so we specify again (the value is retained if
# previously set).
ARG CUDA

#Add image labels
LABEL "com.example.vendor"="Kyiv National Taras Shevchenko University"
LABEL org.opencontainers.image.authors="o_gubar@univ.kiev.ua"
LABEL version="0.2"
LABEL description="Docker container image from official https://github.com/uw-ipd/RoseTTAFold2NA repo with altered folder structure to accomodate separate DB mount folder."


# Use bash to support string substitution.
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install --no-install-recommends -y \
        build-essential \
        cmake \
        cuda-command-line-tools-$(cut -f1,2 -d- <<< ${CUDA//./-}) \
        git \
        hmmer \
        kalign \
        tzdata \
        wget \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get autoremove -y \
    && apt-get clean

# Compile HHsuite from source.
RUN git clone --branch v3.3.0 https://github.com/soedinglab/hh-suite.git /tmp/hh-suite \
    && mkdir /tmp/hh-suite/build \
    && pushd /tmp/hh-suite/build \
    && cmake -DCMAKE_INSTALL_PREFIX=/opt/hhsuite .. \
    && make -j 4 && make install \
    && ln -s /opt/hhsuite/bin/* /usr/bin \
    && popd \
    && rm -rf /tmp/hh-suite
    
    
# Clone lcmsmsguy/RosettaFold2NA repo 
RUN git clone https://github.com/lcmsmsguy/RoseTTAFold2NA.git

# Install Miniconda package manager.
RUN wget -q -P /tmp \
  https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh \
    && bash /tmp/Miniconda3-latest-Linux-x86_64.sh -b -p /opt/conda \
    && rm /tmp/Miniconda3-latest-Linux-x86_64.sh

# Install conda packages.
ENV PATH="/opt/conda/bin:$PATH"
RUN conda install -y -c pytorch -c nvidia -c defaults -c conda-forge \
      python=3.10 \
      cudatoolkit==${CUDA_VERSION} \
      pytorch-gpu \
      requests \
      pytorch-cuda=11.7 \
      dglteam/label/cu117::dgl \
      pyg::pyg \
      bioconda::mafft \
      bioconda::blast \
      bioconda::hmmer>=3.3 \
      bioconda::infernal \
      bioconda::cd-hit \
      bioconda::csblast \
      pip \
      && conda clean --all --force-pkgs-dirs --yes

# Install pip packages.
RUN pip3 install --upgrade pip --no-cache-dir \
    && pip install --no-cache-dir -r RoseTTAFold2NA/SE3Transformer/requirements.txt \
    && pip install pandas

#Builds SE3Transformer from supplied sources
WORKDIR /RoseTTAFold2NA/SE3Transformer
RUN python setup.py install

# Download pre-trained weights under network directory
WORKDIR /RoseTTAFold2NA/network
RUN wget https://files.ipd.uw.edu/dimaio/RF2NA_apr23.tgz \
    && tar xvfz RF2NA_apr23.tgz

# Apply OpenMM patch.
#WORKDIR /opt/conda/lib/python3.8/site-packages

# Add SETUID bit to the ldconfig binary so that non-root users can run it.
RUN chmod u+s /sbin/ldconfig.real

# We need to run `ldconfig` first to ensure GPUs are visible, due to some quirk
# with Debian. See https://github.com/NVIDIA/nvidia-docker/issues/1399 for
# details.
# ENTRYPOINT does not support easily running multiple commands, so instead we
# write a shell script to wrap them up.

