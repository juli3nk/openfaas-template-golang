FROM ghcr.io/openfaas/of-watchdog:0.8.4 as watchdog
FROM golang:1.16-alpine3.13 as build

RUN apk --update --no-cache add git

COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

ENV CGO_ENABLED=0

RUN mkdir -p /go/src/handler
WORKDIR /go/src/handler
COPY . .

RUN echo "replace handler/function => ./function" >> function/go.mod \
	&& ln -s function/go.mod go.mod \
	&& ln -s function/go.sum go.sum

# Run a gofmt and exclude all vendored code.
RUN test -z "$(gofmt -l $(find . -type f -name '*.go' -not -path "./vendor/*" -not -path "./function/vendor/*"))" || { echo "Run \"gofmt -s -w\" on your Golang code"; exit 1; }

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

WORKDIR /go/src/handler/function

RUN go test ./... -cover

WORKDIR /go/src/handler
RUN go build --ldflags "-s -w" -a -installsuffix cgo -o handler .

FROM alpine:3.13
# Add non root user and certs
RUN apk --update --no-cache add ca-certificates \
	&& addgroup -S app \
	&& adduser -S -G app app

WORKDIR /home/app

COPY --from=build --chown=app:app /go/src/handler/handler .
COPY --from=build --chown=app:app /usr/bin/fwatchdog      .

USER app

ENV fprocess="./handler"
ENV mode="http"
ENV upstream_url="http://127.0.0.1:8082"
ENV prefix_logs="false"

CMD ["./fwatchdog"]
