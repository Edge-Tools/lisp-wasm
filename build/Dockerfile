FROM emscripten/emsdk:5.0.7 AS builder

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential autotools-dev automake libtool m4 \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

ARG VERSION=26.5.5
ENV VERSION=${VERSION}
ARG SOURCE_SHA256=a01a5bcda8c5b73e59dda3494fd13e5fec5db6aa1dad782c3cc3bb57f1633435
ENV DIRECTORY=ecl-${VERSION}

WORKDIR /src

ADD https://ecl.common-lisp.dev/static/files/release/ecl-${VERSION}.tgz ecl.tgz
RUN echo "${SOURCE_SHA256}  ecl.tgz" | sha256sum -c - \
    && tar -xzf ecl.tgz && rm ecl.tgz

RUN cp -a ${DIRECTORY} ${DIRECTORY}-host
WORKDIR /src/${DIRECTORY}-host
RUN ./configure --prefix=/host-ecl \
                --disable-shared \
                --without-x \
    && make -j"$(nproc)" \
    && make install
ENV ECL_TO_RUN=/host-ecl/bin/ecl

WORKDIR /src/${DIRECTORY}
COPY js ./js

ENV POST_JS=/src/${DIRECTORY}/js/post.js

ENV CC_FOR_BUILD=gcc
ENV BUILD_CC=gcc

ENV LISP_LDFLAGS="\
    -lnodefs.js -lworkerfs.js \
    --post-js ${POST_JS} \
    -s WASM_BIGINT=1 \
    -s ALLOW_MEMORY_GROWTH=1 \
    -s INITIAL_MEMORY=67108864 \
    -s EXIT_RUNTIME=1 \
    -s EXPORTED_RUNTIME_METHODS=callMain,FS,NODEFS,WORKERFS,ENV \
    -s INCOMING_MODULE_JS_API=noInitialRun,wasmBinary,locateFile,preRun,postRun,instantiateWasm,print,printErr,arguments \
    -s INVOKE_RUN=0 \
    -s MODULARIZE=1 \
    -s EXPORT_ES6=1 \
"

RUN sed -i 's/EXEEXT="\.html"/EXEEXT=".mjs"/' src/configure \
    && grep -q 'EXEEXT=".mjs"' src/configure \
    && ! grep -q 'EXEEXT="\.html"' src/configure

RUN emconfigure ./configure \
      --host=wasm32-unknown-emscripten \
      --build=x86_64-pc-linux-gnu \
      --with-cross-config="$(pwd)/src/util/wasm32-unknown-emscripten.cross_config" \
      --prefix=/wasm-ecl \
      --disable-shared \
      --with-tcp=no \
      --with-cmp=no \
      LDFLAGS="${LISP_LDFLAGS}"

RUN emmake make -j"$(nproc)"

RUN set -eu \
    && mkdir -p /dist \
    && ls -la build/bin/ \
    && cp build/bin/ecl.mjs  /dist/lisp.mjs \
    && cp build/bin/ecl.wasm /dist/lisp.wasm \
    && sed -i 's/ecl\.wasm/lisp.wasm/g' /dist/lisp.mjs \
    && grep -q 'lisp\.wasm' /dist/lisp.mjs \
    && ! grep -q 'ecl\.wasm' /dist/lisp.mjs \
    && [ "$(head -c4 /dist/lisp.wasm | od -An -tx1 | tr -d ' \n')" = "0061736d" ] \
    && [ "$(stat -c%s /dist/lisp.wasm)" -gt 1048576 ]

FROM scratch AS exportor
COPY --from=builder /dist /dist
