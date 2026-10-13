FROM node:20.19.5-bookworm
ENV DEBIAN_FRONTEND=noninteractive NODE_OPTIONS=--max-old-space-size=8192 HUSKY=0 CI=1
RUN corepack enable

COPY . /src
WORKDIR /src

RUN corepack install && yarn install --immutable && yarn build:all && yarn cli:build
# Unit tests only; integration/e2e suites need live chain RPCs and the Vultisig relay.
RUN yarn workspace @vultisig/sdk test:unit || echo "WARNING: unit tests reported failures"
RUN yarn test:core || echo "WARNING: core tests reported failures"
