FROM ghcr.io/openfaas/classic-watchdog:0.1.5 as watchdog
FROM golang:1.16-alpine3.13 as builder

# Required to enable Go modules
RUN apk --update --no-cache add git

# Allows you to add additional packages via build-arg
ARG CGO_ENABLED=0
ARG GO111MODULE="on"
ARG GOSUMDB="off"
ARG GOPROXY=""
ARG GOFLAGS=""

ARG NETRC_ENABLED="false"
ARG NETRC_MACHINE="github.com"
ARG NETRC_LOGIN
ARG NETRC_PASSWORD

RUN test "$NETRC_ENABLED" && printf "machine ${NETRC_MACHINE}\nlogin ${NETRC_LOGIN}\npassword ${NETRC_PASSWORD}\n" >> /root/.netrc \
	&& chmod 600 /root/.netrc

COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

ENV CGO_ENABLED=0

WORKDIR /go/src/handler
COPY . .

RUN echo "replace handler/function => ./function" >> function/go.mod \
	&& ln -s function/go.mod go.mod \
	&& ln -s function/go.sum go.sum

# Run a gofmt and exclude all vendored code.
RUN test -z "$(gofmt -l $(find . -type f -name '*.go' -not -path "./vendor/*" -not -path "./function/vendor/*"))" || { echo "Run \"gofmt -s -w\" on your Golang code"; exit 1; }

WORKDIR /go/src/handler/function

RUN CGO_ENABLED=${CGO_ENABLED} go test ./... -cover

WORKDIR /go/src/handler

RUN GOOD=linux CGO_ENABLED=${CGO_ENABLED} \
    go build --ldflags "-s -w" -a -installsuffix cgo -o handler .


FROM alpine:3.13
RUN apk --update --no-cache add ca-certificates \
	&& addgroup -S app \
	&& adduser -S -G app app

WORKDIR /home/app

COPY --from=builder --chown=app:app /usr/bin/fwatchdog      .
COPY --from=builder --chown=app:app /go/src/handler/handler .

USER app

ENV fprocess="./handler"
EXPOSE 8080

HEALTHCHECK --interval=3s CMD [ -e /tmp/.lock ] || exit 1

CMD ["./fwatchdog"]
