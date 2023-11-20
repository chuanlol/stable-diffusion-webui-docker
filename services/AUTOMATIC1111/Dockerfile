FROM alpine/git:2.36.2 as download

COPY clone.sh /clone.sh

# Clone and remove unnecessary files for stable-diffusion repository
RUN . /clone.sh stable-diffusion-stability-ai https://github.com/Stability-AI/stablediffusion.git cf1d67a6fd5ea1aa600c4df58e5b47da45f6bdbf \
  && rm -rf assets data/**/*.png data/**/*.jpg data/**/*.gif

# Clone and remove unnecessary files for CodeFormer repository
RUN . /clone.sh CodeFormer https://github.com/sczhou/CodeFormer.git c5b4593074ba6214284d6acd5f1719b6c5d739af \
  && rm -rf assets inputs

# Clone BLIP repository
RUN . /clone.sh BLIP https://github.com/salesforce/BLIP.git 48211a1594f1321b00f14c9f7a5b4813144b2fb9

# Clone k-diffusion repository
RUN . /clone.sh k-diffusion https://github.com/crowsonkb/k-diffusion.git ab527a9a6d347f364e3d185ba6d714e22d80cb3c

# Clone clip-interrogator repository
RUN . /clone.sh clip-interrogator https://github.com/pharmapsychotic/clip-interrogator 2cf03aaf6e704197fd0dae7c7f96aa59cf1b11c9

# Clone generative-models repository
RUN . /clone.sh generative-models https://github.com/Stability-AI/generative-models 45c443b316737a4ab6e40413d7794a7f5657c19f

# Clone sd-webui-reactor repository
RUN . /clone.sh sd-webui-reactor https://github.com/Gourieff/sd-webui-reactor 4139d167ff1299a8450a3b4bcbc50dec6cd0690d

# Clone gif2gif repository
RUN . /clone.sh gif2gif https://github.com/LonicaMewinsky/gif2gif 5121851e3941c420253f3a20a0f8e49bed86f0bb

FROM pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime

ENV DEBIAN_FRONTEND=noninteractive PIP_PREFER_BINARY=1
ARG USE_CHINA_MIRROR=true
ENV USE_CHINA_MIRROR=${USE_CHINA_MIRROR}

# Use China mirror if specified
RUN if [ "$USE_CHINA_MIRROR" = "true" ]; then \
        sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list; \
        sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list; \
    fi

RUN --mount=type=cache,target=/var/cache/apt \
  apt-get update && \
  # Install required packages
  apt-get install -y fonts-dejavu-core rsync git jq moreutils aria2 \
  # Install additional packages required by extensions
  ffmpeg libglfw3-dev libgles2-mesa-dev pkg-config libcairo2 libcairo2-dev build-essential


WORKDIR /
RUN --mount=type=cache,target=/root/.cache/pip \
  git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git && \
  cd stable-diffusion-webui && \
  git reset --hard 4afaaf8a020c1df457bcf7250cb1c7f609699fa7 && \
  pip install -r requirements_versions.txt


ENV ROOT=/stable-diffusion-webui

# Copy repositories from the download stage
COPY --from=download /repositories/ ${ROOT}/repositories/

# Copy clip-interrogator data to interrogate directory
RUN mkdir ${ROOT}/interrogate && cp ${ROOT}/repositories/clip-interrogator/clip_interrogator/data/* ${ROOT}/interrogate

RUN --mount=type=cache,target=/root/.cache/pip \
  pip install -r ${ROOT}/repositories/CodeFormer/requirements.txt

RUN --mount=type=cache,target=/root/.cache/pip \
  pip install pyngrok xformers \
  git+https://github.com/TencentARC/GFPGAN.git@8d2447a2d918f8eba5a4a01463fd48e45126a379 \
  git+https://github.com/openai/CLIP.git@d50d76daa670286dd6cacf3bcd80b5e4823fc8e1 \
  git+https://github.com/mlfoundations/open_clip.git@bb6e834e9c70d9c27d0dc3ecedeebeaeb1ffad6b


# Note: don't update the sha of previous versions because the install will take forever
# instead, update the repo state in a later step

# TODO: either remove if fixed in A1111 (unlikely) or move to the top with other apt stuff
RUN apt-get -y install libgoogle-perftools-dev && apt-get clean
ENV LD_PRELOAD=libtcmalloc.so

# Install reactor extension which requires restarting container
RUN --mount=type=cache,target=/root/.cache/pip \
  mv ${ROOT}/repositories/sd-webui-reactor ${ROOT}/extensions/ && \
  PYTHONPATH=${ROOT} python ${ROOT}/extensions/sd-webui-reactor/install.py && \
  echo "CUDA" > ${ROOT}/extensions/sd-webui-reactor/last_device.txt

RUN --mount=type=cache,target=/root/.cache/pip \
  mv ${ROOT}/repositories/gif2gif ${ROOT}/extensions/

COPY . /docker

RUN sha256sum -c /docker/checksum.txt --status

# Run info.py script and rename style.css to user.css
RUN \
  # mv ${ROOT}/style.css ${ROOT}/user.css && \
  # one of the ugliest hacks I ever wrote \
  sed -i 's/in_app_dir = .*/in_app_dir = True/g' /opt/conda/lib/python3.10/site-packages/gradio/routes.py && \
  git config --global --add safe.directory '*'

WORKDIR ${ROOT}
ENV NVIDIA_VISIBLE_DEVICES=all
ENV CLI_ARGS="--api --xformers --no-half --theme dark"
EXPOSE 7860
ENTRYPOINT ["/docker/entrypoint.sh"]
CMD python -u webui.py --listen --port 7860 ${CLI_ARGS}
