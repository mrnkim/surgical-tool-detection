# Load base image (already includes drivers and CUDA toolkit)
FROM nvidia/cuda:13.0.1-cudnn-devel-ubuntu24.04

# Environment variables
ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app
# This exposes the video driver libraries to the container, fixing "Cannot load lib..." errors
ENV NVIDIA_DRIVER_CAPABILITIES video,compute,utility

# 1. Install all system dependencies and build tools at once
# This combines your app's needs with the FFmpeg compilation dependencies from the official guide.
RUN apt-get update -qq && apt-get install -y --no-install-recommends \
    # Your app's dependencies
    python3-pip python3-dev python3-venv curl unzip wget libgl1 libglib2.0-0 \
    # FFmpeg build dependencies from the official documentation
    autoconf automake build-essential cmake git-core libass-dev libfreetype6-dev \
    libgnutls28-dev libmp3lame-dev libsdl2-dev libtool libva-dev libvdpau-dev \
    libvorbis-dev libxcb1-dev libxcb-shm0-dev libxcb-xfixes0-dev meson \
    ninja-build pkg-config texinfo yasm zlib1g-dev nasm libunistring-dev \
    libaom-dev libdav1d-dev libx264-dev libx265-dev libvpx-dev libfdk-aac-dev \
    libopus-dev libnuma-dev \
    && rm -rf /var/lib/apt/lists/*

# 2. Compile and install ffnvcodec headers (as per documentation)
RUN git clone https://git.videolan.org/git/ffmpeg/nv-codec-headers.git && \
    cd nv-codec-headers && \
    make install && \
    cd .. && \
    rm -rf nv-codec-headers

# 3. Compile and install FFmpeg from source with all features
# NOTE: This step will take a long time on the first build!
RUN git clone https://git.ffmpeg.org/ffmpeg.git ffmpeg/ && \
    cd ffmpeg && \
    ./configure \
        --enable-gpl \
        --enable-nonfree \
        --enable-cuda-nvcc \
        --enable-libx264 \
        --enable-libx265 \
        --enable-libvpx \
        --enable-libfdk-aac \
        --enable-libopus \
        --enable-libaom \
        --enable-libass \
        --extra-cflags="-I/usr/local/cuda/include" \
        --extra-ldflags="-L/usr/local/cuda/lib64" \
        --nvccflags="-gencode arch=compute_75,code=sm_75" \
        --disable-static \
        --enable-shared && \
    make -j$(nproc) && \
    make install && \
    cd .. && \
    rm -rf ffmpeg

# --- The rest of your Dockerfile remains the same ---

# Install MediaMTX and Cloudflared
RUN curl -L -o mediamtx.tar.gz https://github.com/bluenviron/mediamtx/releases/download/v1.6.0/mediamtx_v1.6.0_linux_amd64.tar.gz && \
    tar -xzf mediamtx.tar.gz && \
    mv mediamtx /usr/local/bin/ && \
    rm mediamtx.tar.gz

RUN curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb && \
    dpkg -i cloudflared.deb && \
    rm cloudflared.deb

# Set working directory
WORKDIR /app

# Create virtual environment
RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copy application code
COPY . .

# Install other Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Create necessary directories
RUN mkdir -p temp

# Make sure the script is executable
RUN chmod +x main.py

# Expose ports
EXPOSE 8000 8554 1935 8888

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Run the application
CMD ["python3", "main.py"]