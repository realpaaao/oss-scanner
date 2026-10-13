FROM golang:1.25.1-bookworm
ENV DEBIAN_FRONTEND=noninteractive CGO_ENABLED=1 CC=clang
RUN apt-get update && apt-get install -y --no-install-recommends clang redis-server \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

# go-wrappers ships the prebuilt DKLS/Schnorr/ML-DSA shared libraries inside the module; the runtime needs them on the
# loader path.
RUN go mod download && go build -gcflags=all="-N -l" ./... \
    && go build -o /usr/local/bin/vultiserver ./cmd/vultisigner \
    && echo "$(go list -m -f '{{.Dir}}' github.com/vultisig/go-wrappers)/includes/linux" > /etc/ld.so.conf.d/go-wrappers.conf \
    && ldconfig
RUN go vet ./... >/dev/null 2>&1 || true
RUN (redis-server --daemonize yes && go test ./... -timeout 600s) || echo "WARNING: go test reported failures"
