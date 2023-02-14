FROM golang:alpine
RUN \
  apk add --no-cache \
    graphviz \
    git \
    bash \
  && go get -u github.com/google/pprof 

WORKDIR $GOPATH/bin
ENTRYPOINT /bin/bash

