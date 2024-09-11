# Steps for building kong v3.3.0 on s390x
### Pre-reqs
  1. Build machine used for building the following components are RHEL 8.8/Ubuntu 20.04
  2. Install the required dependencies, bazel buildtools, python3 and  GO lang
     
 ### Install dependencies
   For Ubuntu/Debian:
   
   ```
   export DEBIAN_FRONTEND=noninteractive
   apt update -y
   apt install -y \
    automake \
    build-essential \
    curl \
    file \
    git \
    libyaml-dev \
    libprotobuf-dev \
    m4 \
    perl \
    pkg-config \
    procps \
    zip unzip \
    valgrind \
    zlib1g-dev \
    wget \
    cmake \
    openjdk-11-jdk
   
   ```

  Fedora/RHEL:
  
  Enable codeReady base for rhel8 with below command
  
  ```
  subscription-manager repos --enable codeready-builder-for-rhel-8-s390x-rpms
  dnf install -y \
    automake \
    gcc \
    gcc-c++ \
    git \
    libyaml-devel \
    make \
    patch \
    perl \
    perl-IPC-Cmd \
    protobuf-devel \
    zip unzip \
    valgrind \
    valgrind-devel \
    zlib-devel \
    wget \
    cmake \
    java-11-openjdk-devel \
    tzdata-java \
    curl \
    file \
    libffi-devel
            
  ```

### Build steps for kong 3.3.0 from source
1. Set below environment variables

   ```
   export JAVA_HOME=$(compgen -G '/usr/lib/jvm/java-11-openjdk-*')
   export JRE_HOME=${JAVA_HOME}/jre
   export PATH=${JAVA_HOME}/bin:$PATH
   wdir=`pwd`
   PACKAGE_NAME=kong
   PACKAGE_VERSION=${1:-3.3.0}
   PACKAGE_URL=https://github.com/kong/kong/
   PYTHON_VERSION=3.10.2
   ```
2. Install Python3 from source and export the PATH
   
   ```
   wget https://www.python.org/ftp/python/${PYTHON_VERSION}/Python-${PYTHON_VERSION}.tgz
   tar xzf Python-${PYTHON_VERSION}.tgz
   rm -rf Python-${PYTHON_VERSION}.tgz
   cd Python-${PYTHON_VERSION}
   ./configure --enable-shared --with-system-ffi --with-computed-gotos --enable-loadable-sqlite-extensions
   make -j ${nproc}
   make altinstall
   ln -sf $(which python3.10) /usr/bin/python3
   ln -sf $(which pip3.10) /usr/bin/pip3
   ln -s /usr/share/pyshared/lsb_release.py /usr/local/lib/python3.10/site-packages/lsb_release.py
   export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$wdir/Python-3.10.2/lib
   export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib
   python3 -V && pip3 -V
   ```
3. Clone the Kong source code and get the required Bazel version to build.

   ```
   cd $wdir
   git clone ${PACKAGE_URL}
   cd ${PACKAGE_NAME} && git checkout ${PACKAGE_VERSION}
   BAZEL_VERSION=$(cat .bazelversion)
   ```
4. Build and setup required version of Bazel for s390x

   ```
   cd $wdir
   mkdir bazel
   cd bazel
   wget https://github.com/bazelbuild/bazel/releases/download/${BAZEL_VERSION}/bazel-${BAZEL_VERSION}-dist.zip
   unzip bazel-${BAZEL_VERSION}-dist.zip
   rm -rf bazel-${BAZEL_VERSION}-dist.zip
   ./compile.sh
   export PATH=$PATH:$wdir/bazel/output
   ```
5. Install rust and cross with below commands

   ```
   curl https://sh.rustup.rs -sSf | sh -s -- -y && source ~/.cargo/env
   cargo install cross --version 0.2.1
   ```
6. Install Golang for s390x ( version should be > go1.22.0)

   ```
   cd $wdir
   wget https://go.dev/dl/go1.23.1.linux-s390x.tar.gz
   rm -rf /usr/local/go && tar -C /usr/local -xzf go1.23.1.linux-s390x.tar.gz
   rm -rf go1.23.1.linux-s390x.tar.gz
   export PATH=$PATH:/usr/local/go/bin
   go version
   go install github.com/go-task/task/v3/cmd/task@latest
   export PATH=$PATH:$HOME/go/bin
   ```
7. Apply the patch for kong by coping below content to "kong-3.3.0.patch" and build kong for s390x

   ```
    diff --git a/Makefile b/Makefile
    index 917e0070b..4ce00f90f 100644
    --- a/Makefile
    +++ b/Makefile
    @@ -86,7 +86,6 @@ install-dev-rocks: build-venv
     dev: build-venv install-dev-rocks bin/grpcurl
     
     build-release: check-bazel
    -	$(BAZEL) build clean --expunge
     	$(BAZEL) build //build:kong --verbose_failures --config release
     
     package/deb: check-bazel build-release
    diff --git a/build/nfpm/repositories.bzl b/build/nfpm/repositories.bzl
    index cc719072e..c3bbf0eb2 100644
    --- a/build/nfpm/repositories.bzl
    +++ b/build/nfpm/repositories.bzl
    @@ -15,6 +15,8 @@ def _nfpm_release_select_impl(ctx):
             os_arch = "arm64"
         elif os_arch == "amd64":
             os_arch = "x86_64"
    +    elif os_arch == "s390x":
    +        os_arch = "s390x"
         else:
             fail("Unsupported arch %s" % os_arch)
     
    @@ -38,13 +40,14 @@ def nfpm_repositories():
         npfm_matrix = [
             ["linux", "x86_64", "4c63031ddbef198e21c8561c438dde4c93c3457ffdc868d7d28fa670e0cc14e5"],
             ["linux", "arm64", "2af1717cc9d5dcad5a7e42301dabc538acf5d12ce9ee39956c66f30215311069"],
    +        ["linux", "s390x", "4c63031ddbef198e21c8561c438dde4c93c3457ffdc868d7d28fa670e0cc14e5"],
             ["Darwin", "x86_64", "fb3b8ab5595117f621c69cc51db71d481fbe733fa3c35500e1b64319dc8fd5b4"],
             ["Darwin", "arm64", "9ca3ac6e0c4139a9de214f78040d1d11dd221496471696cc8ab5d357850ccc54"],
         ]
         for name, arch, sha in npfm_matrix:
             http_archive(
                 name = "nfpm_%s_%s" % (name, arch),
    -            url = "https://github.com/goreleaser/nfpm/releases/download/v2.23.0/nfpm_2.23.0_%s_%s.tar.gz" % (name, arch),
    +            url = "https://github.com/goreleaser/nfpm/releases/download/v2.23.0/nfpm_2.23.0_%s_x86_64.tar.gz" % (name),
                 sha256 = sha,
                 build_file = "//build/nfpm:BUILD.bazel",
             )
    diff --git a/build/nfpm/rules.bzl b/build/nfpm/rules.bzl
    index d6f5bb94f..14a6ca678 100644
    --- a/build/nfpm/rules.bzl
    +++ b/build/nfpm/rules.bzl
    @@ -11,6 +11,8 @@ def _nfpm_pkg_impl(ctx):
         target_cpu = ctx.attr._cc_toolchain[cc_common.CcToolchainInfo].cpu
         if target_cpu == "k8" or target_cpu == "x86_64" or target_cpu == "amd64":
             target_arch = "amd64"
    +    if target_cpu == "s390" or target_cpu == "s390x":
    +        target_arch = "s390x"
         elif target_cpu == "aarch64" or target_cpu == "arm64":
             target_arch = "arm64"
         else:
    @@ -24,7 +26,7 @@ def _nfpm_pkg_impl(ctx):
         if pkg_ext == "apk":
             pkg_ext = "apk.tar.gz"
     
    -    # create like kong.amd64.deb
    +    # create like kong.<target_arch>.deb package
         out = ctx.actions.declare_file("%s/%s.%s.%s" % (
             ctx.attr.out_dir,
             ctx.attr.pkg_name,
    @@ -41,7 +43,7 @@ def _nfpm_pkg_impl(ctx):
         ctx.actions.run_shell(
             inputs = ctx.files._nfpm_bin,
             mnemonic = "nFPM",
    -        command = "ln -sf %s nfpm-prefix; external/nfpm/nfpm $@" % KONG_VAR["BUILD_DESTDIR"],
    +        command = "ln -sf %s nfpm-prefix; build/nfpm/nfpm $@" % KONG_VAR["BUILD_DESTDIR"],
             arguments = [nfpm_args],
             outputs = [out],
             env = env,
    diff --git a/build/openresty/BUILD.openresty.bazel b/build/openresty/BUILD.openresty.bazel
    index 812ddfb9e..4cfaa90e8 100644
    --- a/build/openresty/BUILD.openresty.bazel
    +++ b/build/openresty/BUILD.openresty.bazel
    @@ -84,6 +84,7 @@ make(
             "-j" + KONG_VAR["NPROC"],
             "install",
         ],
    +    postfix_script = "make clean; ASFLAGS= make all PREFIX=$INSTALLDIR; ASFLAGS= make install PREFIX=$INSTALLDIR",
         visibility = ["//visibility:public"],
     )
    
    diff --git a/kong/pdk/nginx.lua b/kong/pdk/nginx.lua
    index f5715e03e..7d1daa479 100644
    --- a/kong/pdk/nginx.lua
    +++ b/kong/pdk/nginx.lua
    @@ -12,7 +12,7 @@ local arch = ffi.arch
     local ngx  = ngx
     local tonumber = tonumber
     
    -if arch == "x64" or arch == "arm64" then
    +if arch == "x64" or arch == "s390x" or arch == "arm64" then
       ffi.cdef[[
         uint64_t *ngx_stat_active;
         uint64_t *ngx_stat_reading;
   ```

   ```
   cd $wdir/${PACKAGE_NAME}
   git apply $wdir/kong-${PACKAGE_VERSION}.patch
   make build-release > /dev/null 2>&1 || true
   ```
8. Apply the patch for rules_rust repo by coping below content into "kong-3.3.0-rules_rust.patch"
   ```
    diff --git a/rust/platform/triple.bzl b/rust/platform/triple.bzl
    index 6460e2f8..9426eaf4 100644
    --- a/rust/platform/triple.bzl
    +++ b/rust/platform/triple.bzl
    @@ -151,7 +151,7 @@ def get_host_triple(repository_ctx, abi = None):
         # Detect the host's cpu architecture
     
         supported_architectures = {
    -        "linux": ["aarch64", "x86_64"],
    +        "linux": ["aarch64", "x86_64", "s390x"],
             "macos": ["aarch64", "x86_64"],
             "windows": ["aarch64", "x86_64"],
         }
    diff --git a/rust/repositories.bzl b/rust/repositories.bzl
    index 73006c63..4a683b1e 100644
    --- a/rust/repositories.bzl
    +++ b/rust/repositories.bzl
    @@ -43,6 +43,7 @@ DEFAULT_TOOLCHAIN_TRIPLES = {
         "x86_64-pc-windows-msvc": "rust_windows_x86_64",
         "x86_64-unknown-freebsd": "rust_freebsd_x86_64",
         "x86_64-unknown-linux-gnu": "rust_linux_x86_64",
    +    "s390x-unknown-linux-gnu": "rust_linux_s390x",
     }
     
     def rules_rust_dependencies():
   ```

   ```
   pushd $(find $HOME/.cache/bazel -name rules_rust)
   git apply $wdir/kong-${PACKAGE_VERSION}-rules_rust.patch
   ```
9. Build cargo-bazel native binary for s390x
   ```
   cd crate_universe
   cross build --release --locked --bin cargo-bazel --target=s390x-unknown-linux-gnu
   export CARGO_BAZEL_GENERATOR_URL=file://$(pwd)/target/s390x-unknown-linux-gnu/release/cargo-bazel
   popd
   ```
10. Build nfpm native binary for s390x
    ```
    cd $wdir
    git clone https://github.com/goreleaser/nfpm.git
    cd nfpm && git checkout v2.30.1
    task setup
    task build
    NFPM_BIN=$(pwd)/nfpm
    ```
11. Finally build the kong.rpm package for s390x with below commands. This will generate and copy the kong.el9.s390x.rpm into work dir 
    ```
    cd $wdir/${PACKAGE_NAME}
    make package/deb  > /dev/null 2>&1 || true
    cp -f $NFPM_BIN $(find $HOME/.cache/bazel -type d -name nfpm)
    cp -f $NFPM_BIN $wdir/kong/build/nfpm/
    make package/rpm
    cp $(find / -name kong.el8.s390x.rpm) $wdir
    ```
# Build docker image with kong.rpm for s390x
1. Clone https://github.com/Kong/docker-kong.git and checkout 3.3.0 tag

   ```
   cd $wdir
   git clone https://github.com/Kong/docker-kong.git
   cd docker-kong
   git checkout 3.3.0
   ```
3. Replace the existing kong.rpm with above generated kong rpm and start docker build with below command after editing Dockerfile.rpm to set "ARG ASSET=remote" to local

   ```
   mv $wdir/kong.el8.s390x.rpm $wdir/docker-kong/kong.rpm
   docker build -t kong:v3.3.0 -f Dockerfile.rpm
   ```
4. Finally, verify the built image with docker run and check kong version

   ```
   docker run -it localhost/kong:v3.3.0 /bin/bash
   
   [root@kong-371-rhel1 docker-kong]# docker run -it localhost/kong:v3.3.0 /bin/bash
   Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
   [kong@aad4bcc95f52 /]$ kong version
   3.3.0
   ```
   
   

      
      
      
