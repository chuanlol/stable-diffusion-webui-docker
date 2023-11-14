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

# Stage 2: Install xformers
FROM alpine:3.17 as xformers
RUN apk add --no-cache aria2
RUN aria2c -x 5 --dir / --out wheel.whl 'https://github.com/AbdBarho/stable-diffusion-webui-docker/releases/download/6.0.0/xformers-0.0.21.dev544-cp310-cp310-manylinux2014_x86_64-pytorch201.whl'


# Stage 3: Build the final image
FROM python:3.10.9-slim

ENV DEBIAN_FRONTEND=noninteractive PIP_PREFER_BINARY=1
ARG USE_CHINA_MIRROR=true
ENV USE_CHINA_MIRROR=${USE_CHINA_MIRROR}

# Use China mirror if specified
RUN if [ "$USE_CHINA_MIRROR" = "true" ]; then \
        sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list; \
    fi

RUN --mount=type=cache,target=/var/cache/apt \
  apt-get update && \
  # Install required packages
  apt-get install -y fonts-dejavu-core rsync git jq moreutils aria2 \
  # Install additional packages required by extensions
  ffmpeg libglfw3-dev libgles2-mesa-dev pkg-config libcairo2 libcairo2-dev build-essential


RUN --mount=type=cache,target=/cache --mount=type=cache,target=/root/.cache/pip \
  aria2c -x 5 --dir /cache --out torch-2.0.1-cp310-cp310-linux_x86_64.whl -c \
  https://download.pytorch.org/whl/cu118/torch-2.0.1%2Bcu118-cp310-cp310-linux_x86_64.whl && \
  pip install /cache/torch-2.0.1-cp310-cp310-linux_x86_64.whl torchvision --index-url https://download.pytorch.org/whl/cu118


RUN --mount=type=cache,target=/root/.cache/pip \
  git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git && \
  cd stable-diffusion-webui && \
  git reset --hard 5ef669de080814067961f28357256e8fe27544f4 && \
  pip install -r requirements_versions.txt

RUN --mount=type=cache,target=/root/.cache/pip  \
  --mount=type=bind,from=xformers,source=/wheel.whl,target=/xformers-0.0.21.dev544-cp310-cp310-manylinux2014_x86_64.whl \
  pip install /xformers-0.0.21.dev544-cp310-cp310-manylinux2014_x86_64.whl

ENV ROOT=/stable-diffusion-webui

# Copy repositories from the download stage
COPY --from=download /repositories/ ${ROOT}/repositories/

# Copy clip-interrogator data to interrogate directory
RUN mkdir ${ROOT}/interrogate && cp ${ROOT}/repositories/clip-interrogator/clip_interrogator/data/* ${ROOT}/interrogate

RUN --mount=type=cache,target=/root/.cache/pip \
  pip install -r ${ROOT}/repositories/CodeFormer/requirements.txt

RUN --mount=type=cache,target=/root/.cache/pip \
  pip install pyngrok \
  git+https://github.com/TencentARC/GFPGAN.git@8d2447a2d918f8eba5a4a01463fd48e45126a379 \
  git+https://github.com/openai/CLIP.git@d50d76daa670286dd6cacf3bcd80b5e4823fc8e1 \
  git+https://github.com/mlfoundations/open_clip.git@bb6e834e9c70d9c27d0dc3ecedeebeaeb1ffad6b


# Note: don't update the sha of previous versions because the install will take forever
# instead, update the repo state in a later step

# TODO: either remove if fixed in A1111 (unlikely) or move to the top with other apt stuff
RUN apt-get -y install libgoogle-perftools-dev && apt-get clean
ENV LD_PRELOAD=libtcmalloc.so

ARG SHA=4afaaf8a020c1df457bcf7250cb1c7f609699fa7
RUN --mount=type=cache,target=/root/.cache/pip \
  cd stable-diffusion-webui && \
  git fetch && \
  git reset --hard ${SHA} && \
  pip install -r requirements_versions.txt

# Install reactor extension which requires restarting container
RUN --mount=type=cache,target=/root/.cache/pip \
  mv ${ROOT}/repositories/sd-webui-reactor ${ROOT}/extensions/ && \
  PYTHONPATH=${ROOT} python ${ROOT}/extensions/sd-webui-reactor/install.py && \
  echo "CUDA" > ${ROOT}/extensions/sd-webui-reactor/last_device.txt

COPY . /docker

RUN sha256sum -c /docker/checksum.txt --status

# Run info.py script and rename style.css to user.css
RUN \
  python3 /docker/info.py ${ROOT}/modules/ui.py && \
  mv ${ROOT}/style.css ${ROOT}/user.css && \
  # Apply a hack to routes.py file
  sed -i 's/in_app_dir = .*/in_app_dir = True/g' /usr/local/lib/python3.10/site-packages/gradio/routes.py && \
  git config --global --add safe.directory '*'

WORKDIR ${ROOT}
ENV NVIDIA_VISIBLE_DEVICES=all
ENV CLI_ARGS="--xformers --no-half --theme dark"
EXPOSE 7860
ENTRYPOINT ["/docker/entrypoint.sh"]
CMD python -u webui.py --listen --port 7860 ${CLI_ARGS}
