FROM ghcr.io/openfaas/classic-watchdog:0.1.5 as watchdog
FROM golang:1.17-alpine3.14 as builder

RUN apk --update --no-cache add git

COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

ARG NETRC_ENABLED="false"
ARG NETRC_MACHINE="github.com"
ARG NETRC_LOGIN
ARG NETRC_PASSWORD

RUN test "$NETRC_ENABLED" \
	&& printf "machine ${NETRC_MACHINE}\nlogin ${NETRC_LOGIN}\npassword ${NETRC_PASSWORD}\n" >> /root/.netrc \
	&& chmod 600 /root/.netrc

ARG GO111MODULE="on"
ARG GOSUMDB="off"
ARG GOPROXY=""
ARG GOPRIVATE=""
ARG GOFLAGS=""

ENV CGO_ENABLED=0

RUN mkdir -p /go/src/handler
WORKDIR /go/src/handler
COPY . .

RUN echo -e "replace handler/function => ./function\nrequire handler/function v0.0.0" >> function/go.mod \
	&& ln -s function/go.mod go.mod \
	&& ln -s function/go.sum go.sum

WORKDIR /go/src/handler/function

# Run a gofmt and exclude all vendored code.
RUN test -z "$(gofmt -l $(find . -type f -name '*.go' -not -path "./vendor/*"))" || { echo "Run \"gofmt -s -w\" on your Golang code"; exit 1; }

RUN go test ./... -cover

WORKDIR /go/src/handler

RUN go build --ldflags "-s -w" -a -installsuffix cgo -o handler .


FROM alpine:3.14

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
