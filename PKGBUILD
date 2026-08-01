# Maintainer: LeonRein
: ${aur_llamacpp_build_universal:=false}
# CUDA architectures to compile for. "native" targets only the GPUs present in
# this machine, which is much faster than the upstream default arch list.
# Ignored for universal builds. Override with e.g. aur_llamacpp_cuda_arch="86;89"
: ${aur_llamacpp_cuda_arch:=native}
pkgname=llama-cpp-turboquant-cuda-git
_pkgname="${pkgname%-cuda-git}"
pkgver=feature.turboquant.kv.cache.b9999.8a891f4.r0.8a891f4b5
pkgrel=1
pkgdesc="Port of Facebook's LLaMA model in C/C++ with TurboQuant KV-cache compression (NVIDIA CUDA optimizations)"
arch=(x86_64 aarch64)
url='https://github.com/TheTom/llama-cpp-turboquant'
license=('MIT')
backup=('etc/conf.d/llama.cpp')
depends=(
  cuda
  curl
  gcc-libs
  glibc
  nvidia-utils
  openssl
)
makedepends=(
  cmake
  cudnn
  gcc15   # (CUDA does not yet support GCC 16)
  git
  ninja
  npm     # builds the embedded server Web UI
)
optdepends=(
'ccache: greatly reduce package re-build time'
'nccl: needed for multi-GPU parallelism'
'python-numpy: needed for convert_hf_to_gguf.py'
'python-safetensors: needed for convert_hf_to_gguf.py'
'python-sentencepiece: needed for convert_hf_to_gguf.py'
'python-pytorch: needed for convert_hf_to_gguf.py'
'python-transformers: needed for convert_hf_to_gguf.py'
'rdma-core: RDMA transport for RPC backend'
)
# Note: This package provides libggml (with CUDA) and libggml-cuda-git to support
# downstream packages like whisper.cpp-cuda that require CUDA-enabled GGML backends.
provides=(
  "${_pkgname}"
  libggml-cuda-git
  libggml
  libggml.so
  ggml
)
conflicts=(
  "${_pkgname}"
  libggml
  ggml
)
source=(
"git+https://github.com/TheTom/llama-cpp-turboquant.git"
llama.cpp.conf
llama.cpp.service
)
sha256sums=('SKIP'
            'SKIP'
            'SKIP')
b2sums=('SKIP'
        'SKIP'
        'SKIP')

pkgver() {
  cd "${_pkgname}" || exit
  printf "%s" "$(git describe --long --tags | sed 's/\([^-]*-\)g/r\1/;s/-/./g')"
}

build() {
  if ! type -P nvcc &>/dev/null && [[ -d /opt/cuda/bin ]]; then
    export PATH="/opt/cuda/bin:$PATH"
  fi

  # Derived here rather than in prepare() so they are still correct when the
  # prepare step is skipped (makepkg -e / --noprepare).
  local _commit_id _build_number
  _commit_id=$(git -C "${_pkgname}" rev-parse HEAD)
  _build_number=$(git -C "${_pkgname}" rev-list --count HEAD)

  # Use GCC 15 as host compiler for nvcc (CUDA does not yet support GCC 16)
  # Override via: aur_llamacpp_cmakeopts="-DCMAKE_CUDA_HOST_COMPILER=/usr/bin/g++-XX"
  local _nvcc_host_cxx="${CUDAHOSTCXX:-/usr/bin/g++-15}"

  local _cmake_options=(
    -G Ninja
    -B build
    -S "${_pkgname}"
    -DCMAKE_BUILD_TYPE=Release
    -DCMAKE_INSTALL_PREFIX='/usr'
    -DCMAKE_CUDA_HOST_COMPILER="${_nvcc_host_cxx}"
    -DBUILD_SHARED_LIBS=ON
    -DLLAMA_BUILD_TESTS=OFF
    -DLLAMA_USE_SYSTEM_GGML=OFF
    -DGGML_ALL_WARNINGS=OFF
    -DGGML_ALL_WARNINGS_3RD_PARTY=OFF
    -DGGML_BUILD_EXAMPLES=OFF
    -DGGML_BUILD_TESTS=OFF
    -DGGML_OPENMP=ON
    -DGGML_LTO=ON
    -DGGML_RPC=ON
    -DGGML_CUDA=ON
    -DGGML_CUDA_FA_ALL_QUANTS=ON
    -DGGML_CUDNN=ON
    -DGGML_CUDA_COMPRESSION_MODE=speed
    -DLLAMA_BUILD_SERVER=ON
    # Build the Web UI from source with npm. The prebuilt HF bucket
    # (ggml-org/llama-ui) does not ship loading.html, which this fork requires
    # (tools/ui/embed.cpp), so the prebuilt path silently yields a UI-less
    # server that answers / with a 404. Keep PREBUILT off so UI breakage is loud.
    -DLLAMA_BUILD_UI=ON
    -DLLAMA_USE_PREBUILT_UI=OFF
    -DLLAMA_BUILD_NUMBER="${_build_number}"
    -DLLAMA_BUILD_COMMIT="${_commit_id}"
    -DLLAMA_OPENSSL=ON
    -Wno-dev
  )

  if [[ ${aur_llamacpp_build_universal} == true ]]; then
    echo "Building universal binary [aur_llamacpp_build_universal == true]"
    _cmake_options+=(
      -DGGML_BACKEND_DL=ON
      -DGGML_NATIVE=OFF
      -DGGML_CPU_ALL_VARIANTS=ON
    )
  else
    # we lose GGML_NATIVE_DEFAULT due to how makepkg includes
    # $SOURCE_DATE_EPOCH in ENV
    _cmake_options+=(
      -DGGML_NATIVE=ON
      -DCMAKE_CUDA_ARCHITECTURES="${aur_llamacpp_cuda_arch}"
    )
  fi

  # Allow user-specified additional flags
  if [[ -n "${aur_llamacpp_cmakeopts:-}" ]]; then
    echo "Applying custom CMake options: ${aur_llamacpp_cmakeopts}"
    # shellcheck disable=SC2206 # intentional word splitting
    _cmake_options+=(${aur_llamacpp_cmakeopts})
  fi

  cmake "${_cmake_options[@]}"
  cmake --build build
}

package() {
  DESTDIR="${pkgdir}" cmake --install build
  install -Dm644 "${_pkgname}/LICENSE" "${pkgdir}/usr/share/licenses/${pkgname}/LICENSE"
  install -Dm644 "llama.cpp.conf" "${pkgdir}/etc/conf.d/llama.cpp"
  install -Dm644 "llama.cpp.service" "${pkgdir}/usr/lib/systemd/system/llama.cpp.service"
}
