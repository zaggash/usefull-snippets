FROM golang:alpine
RUN \
  apk add --no-cache \
    graphviz \
    git \
    bash \
  && go install github.com/google/pprof@latest

WORKDIR $GOPATH/bin
ENTRYPOINT /bin/bash

