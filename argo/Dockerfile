####################################################################################################
# Builder image
# Initial stage which pulls prepares build dependencies and CLI tooling we need for our final image
# Also used as the image in CI jobs so needs all dependencies
####################################################################################################
FROM golang:1.11.4 as builder

RUN apt-get update && apt-get install -y \
    git \
    make \
    wget \
    gcc \
    zip && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

WORKDIR /tmp

# Install docker
ENV DOCKER_CHANNEL stable
ENV DOCKER_VERSION 18.09.1
RUN wget -O docker.tgz "https://download.docker.com/linux/static/${DOCKER_CHANNEL}/x86_64/docker-${DOCKER_VERSION}.tgz" && \
    tar --extract --file docker.tgz --strip-components 1 --directory /usr/local/bin/ && \
    rm docker.tgz

# Install dep
ENV DEP_VERSION=0.5.0
RUN wget https://github.com/golang/dep/releases/download/v${DEP_VERSION}/dep-linux-amd64 -O /usr/local/bin/dep && \
    chmod +x /usr/local/bin/dep

# Install gometalinter
ENV GOMETALINTER_VERSION=2.0.12
RUN curl -sLo- https://github.com/alecthomas/gometalinter/releases/download/v${GOMETALINTER_VERSION}/gometalinter-${GOMETALINTER_VERSION}-linux-amd64.tar.gz | \
    tar -xzC "$GOPATH/bin" --exclude COPYING --exclude README.md --strip-components 1 -f- && \
    ln -s $GOPATH/bin/gometalinter $GOPATH/bin/gometalinter.v2


####################################################################################################
# argoexec-base
# Used as the base for both the release and development version of argoexec
####################################################################################################
FROM debian:9.6-slim as argoexec-base
# NOTE: keep the version synced with https://storage.googleapis.com/kubernetes-release/release/stable.txt
ENV KUBECTL_VERSION=1.13.1
RUN apt-get update && \
    apt-get install -y curl jq procps git tar mime-support && \
    rm -rf /var/lib/apt/lists/* && \
    curl -L -o /usr/local/bin/kubectl -LO https://storage.googleapis.com/kubernetes-release/release/v${KUBECTL_VERSION}/bin/linux/amd64/kubectl && \
    chmod +x /usr/local/bin/kubectl
COPY --from=builder /usr/local/bin/docker /usr/local/bin/


####################################################################################################
# Argo Build stage which performs the actual build of Argo binaries
####################################################################################################
FROM builder as argo-build
#ENV ARGO_VERSION=2.2.1
ENV PACKAGE=github.com/argoproj/argo

# A dummy directory is created under $GOPATH/src/dummy so we are able to use dep
# to install all the packages of our dep lock file
RUN mkdir -p ${GOPATH}/src/dummy \
  && curl -sSLf -o ${GOPATH}/src/dummy/Gopkg.toml https://raw.githubusercontent.com/argoproj/argo/master/Gopkg.toml \
  && curl -sSLf -o ${GOPATH}/src/dummy/Gopkg.lock https://raw.githubusercontent.com/argoproj/argo/master/Gopkg.lock
#  && curl -sSLf -o ${GOPATH}/src/dummy/Gopkg.toml https://raw.githubusercontent.com/argoproj/argo/v${ARGO_VERSION}/Gopkg.toml \
#  && curl -sSLf -o ${GOPATH}/src/dummy/Gopkg.lock https://raw.githubusercontent.com/argoproj/argo/v${ARGO_VERSION}/Gopkg.lock

#COPY Gopkg.toml ${GOPATH}/src/dummy/Gopkg.toml
#COPY Gopkg.lock ${GOPATH}/src/dummy/Gopkg.lock

RUN cd ${GOPATH}/src/dummy && \
    dep ensure -vendor-only && \
    mv vendor/* ${GOPATH}/src/ && \
    rmdir vendor

#RUN curl -sSLf https://${PACKAGE}/archive/v${ARGO_VERSION}.tar.gz | \
RUN curl -sSLf https://${PACKAGE}/archive/master.tar.gz | \
  tar zxf - -C /tmp/ \
  && mkdir -p ${GOPATH}/src/github.com/argoproj/ \
  && mv argo-master ${GOPATH}/src/${PACKAGE}
#  && mv argo-${ARGO_VERSION} ${GOPATH}/src/${PACKAGE}


# Perform the build
WORKDIR /go/src/github.com/argoproj/argo
COPY . .
ARG MAKE_TARGET="controller executor cli-linux-amd64"
RUN make $MAKE_TARGET


####################################################################################################
# argoexec
####################################################################################################
FROM argoexec-base as argoexec
COPY --from=argo-build /go/src/github.com/argoproj/argo/dist/argoexec /usr/local/bin/


####################################################################################################
# workflow-controller
####################################################################################################
FROM scratch as workflow-controller
COPY --from=argo-build /go/src/github.com/argoproj/argo/dist/workflow-controller /bin/
ENTRYPOINT [ "workflow-controller" ]


####################################################################################################
# argocli
####################################################################################################
FROM scratch as argocli
COPY --from=argo-build /go/src/github.com/argoproj/argo/dist/argo-linux-amd64 /bin/argo
ENTRYPOINT [ "argo" ]
