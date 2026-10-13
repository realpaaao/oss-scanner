FROM node:24.14.1-bookworm
ENV DEBIAN_FRONTEND=noninteractive NODE_OPTIONS=--max-old-space-size=8192 HUSKY=0 CI=1 \
    PATH=/usr/local/go/bin:/root/go/bin:$PATH
COPY --from=golang:1.24.2-bookworm /usr/local/go /usr/local/go
RUN corepack enable

COPY . /src
WORKDIR /src

# Desktop is a Wails app (Go backend + React frontend); the browser extension is a Vite MV3 build. Both share core/ and lib/.
RUN corepack install && yarn install --immutable && yarn build:extension
RUN go mod download && (go build ./mediator/... ./relay/... ./storage/... ./tss/... ./utils/... || echo "WARNING: go build failed")
RUN yarn test || echo "WARNING: core/lib tests reported failures"
RUN yarn workspace @clients/extension test || echo "WARNING: extension tests reported failures"
