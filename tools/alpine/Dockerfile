FROM alpine:3.6 AS mirror

# update base image
RUN apk update && apk upgrade -a

# Copy Dockerfile so we can include it in the hash
COPY Dockerfile /Dockerfile
COPY packages /tmp/

# mirror packages
RUN mkdir -p /mirror/$(uname -m) && \
   apk fetch --recursive -o /mirror/$(uname -m) $(apk info; cat /tmp/packages)

# add the tools for WireGuard, since the kernel module is now included, but from edge/testing
RUN apk fetch --recursive -o /mirror/$(uname -m) -X http://dl-cdn.alpinelinux.org/alpine/edge/testing -U wireguard-tools

# install abuild for signing
RUN apk add --no-cache abuild

# install a new key into /etc/apk/keys
RUN abuild-keygen -a -i -n

# index the new repo
RUN apk index --rewrite-arch $(uname -m) -o /mirror/$(uname -m)/APKINDEX.unsigned.tar.gz /mirror/$(uname -m)/*.apk

# sign the index
RUN cp /mirror/$(uname -m)/APKINDEX.unsigned.tar.gz /mirror/$(uname -m)/APKINDEX.tar.gz
RUN abuild-sign /mirror/$(uname -m)/APKINDEX.tar.gz

# fetch OVMF for qemu EFI boot (this is not added as a package)
RUN apk add -X http://dl-cdn.alpinelinux.org/alpine/edge/community ovmf

# set this as our repo but keep a copy of the upstream for downstream use
RUN mv /etc/apk/repositories /etc/apk/repositories.upstream && echo "/mirror" > /etc/apk/repositories && apk update

# add Go validation tools
COPY go-compile.sh /go/bin/
RUN apk add --no-cache git go musl-dev
ENV GOPATH=/go PATH=$PATH:/go/bin
RUN go get -u github.com/golang/lint/golint
RUN go get -u github.com/gordonklaus/ineffassign
RUN go get -u github.com/LK4D4/vndr

FROM alpine:3.6

COPY --from=mirror /etc/apk/repositories /etc/apk/repositories
COPY --from=mirror /etc/apk/repositories.upstream /etc/apk/repositories.upstream
COPY --from=mirror /etc/apk/keys /etc/apk/keys/
COPY --from=mirror /mirror /mirror/
COPY --from=mirror /go/bin /go/bin/
COPY --from=mirror /usr/share/ovmf/bios.bin /usr/share/ovmf/bios.bin
COPY --from=mirror /Dockerfile /Dockerfile

RUN apk update && apk upgrade -a

# v1.0.0-alpha1 plus https://github.com/containerd/containerd/pull/1141
ENV CONTAINERD_REPO=https://github.com/ijc/containerd.git
ENV CONTAINERD_COMMIT=d42cb88ba2b08d2ca6c8c017d629b394bf1dd08c
