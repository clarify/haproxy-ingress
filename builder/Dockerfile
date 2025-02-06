# Build stage (multi-arch compatible)
FROM golang:1.19.13-alpine AS builder

ARG GIT_TAG
WORKDIR /src
RUN apk --no-cache add git

COPY . .

RUN GIT_TAG=${GIT_TAG:-latest} \
 && GIT_COMMIT=git-$(git rev-parse --short HEAD) \
 && GIT_REPO=$(git config --get remote.origin.url) \
 && VERSION_PKG=github.com/jcmoraisjr/haproxy-ingress/pkg/version \
 && CGO_ENABLED=0 GOOS=linux GOARCH=$(go env GOARCH) go build \
    -ldflags "-s -w -X ${VERSION_PKG}.RELEASE=${GIT_TAG} -X ${VERSION_PKG}.COMMIT=${GIT_COMMIT} -X ${VERSION_PKG}.REPO=${GIT_REPO}" \
    -o haproxy-ingress pkg/main.go

# Final image
FROM haproxy:2.4.26-alpine

USER root

RUN apk upgrade --no-cache && apk --no-cache add \
    socat \
    openssl \
    lua5.3 \
    lua-json4 \
    dumb-init

COPY rootfs/ /
COPY --from=builder /src/haproxy-ingress /haproxy-ingress-controller

# Set up user properly
RUN deluser haproxy \
 && addgroup -g 1001 haproxy \
 && adduser -u 1001 -G haproxy -D -s /bin/false haproxy \
 && mkdir -p /var/empty /etc/haproxy /var/lib/haproxy /var/run/haproxy \
 && chown -R haproxy:haproxy /etc/haproxy /var/lib/haproxy /var/run/haproxy \
 && chmod 0 /var/empty

STOPSIGNAL SIGTERM

# dumb-init reaps the old haproxy process in the embedded, non master-worker mode,
# after receiving SIGUSR1, avoiding it to become a zombie.
ENTRYPOINT ["/usr/bin/dumb-init", "--", "/start.sh"]
