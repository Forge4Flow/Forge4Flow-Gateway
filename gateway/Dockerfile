FROM --platform=${BUILDPLATFORM:-linux/amd64} ghcr.io/openfaas/license-check:0.4.1 as license-check

FROM --platform=${BUILDPLATFORM:-linux/amd64} golang:1.20 as build

ENV GO111MODULE=on
ENV CGO_ENABLED=0

ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETOS
ARG TARGETARCH

ARG GIT_COMMIT
ARG VERSION

COPY --from=license-check /license-check /usr/bin/

WORKDIR /go/src/github.com/forge4flow/forge4flow-gateway/gateway

COPY vendor         vendor
COPY go.mod         go.mod
COPY go.sum         go.sum

COPY handlers       handlers
COPY metrics        metrics
COPY requests       requests

COPY types          types
COPY plugin         plugin
COPY version        version
COPY scaling        scaling
COPY pkg            pkg
COPY main.go        .

RUN license-check -path ./ --verbose=false "BoiseITGuru" "Forge4Flow Authors" "Forge4Flow Author(s)"

# Run a gofmt and exclude all vendored code.
RUN test -z "$(gofmt -l $(find . -type f -name '*.go' -not -path "./vendor/*"))"
RUN go test $(go list ./... | grep -v integration | grep -v /vendor/ | grep -v /template/) -cover

RUN CGO_ENABLED=${CGO_ENABLED} GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build --ldflags "-s -w \
    -X \"github.com/forge4flow/forge4flow-gateway/gateway/version.GitCommitSHA=${GIT_COMMIT}\" \
    -X \"github.com/forge4flow/forge4flow-gateway/gateway/version.Version=${VERSION}\" \
    -X github.com/forge4flow/forge4flow-gateway/gateway/types.Arch=${TARGETARCH}" \
    -a -installsuffix cgo -o gateway .

FROM --platform=${TARGETPLATFORM:-linux/amd64} alpine:3.18.3 as ship

LABEL org.label-schema.license="MIT" \
    org.label-schema.vcs-url="https://github.com/forge4flow/forge4flow-gateway" \
    org.label-schema.vcs-type="Git" \
    org.label-schema.name="openfaas/faas" \
    org.label-schema.vendor="openfaas" \
    org.label-schema.docker.schema-version="1.0"

LABEL org.opencontainers.image.source=https://github.com/forge4flow/forge4flow-gateway

RUN addgroup -S app \
    && adduser -S -g app app \
    && apk add --no-cache ca-certificates

WORKDIR /home/app

EXPOSE 8080
EXPOSE 8082
ENV http_proxy      ""
ENV https_proxy     ""

COPY --from=build /go/src/github.com/forge4flow/forge4flow-gateway/gateway/gateway    .
COPY assets     assets

ARG TARGETPLATFORM

RUN if [ "$TARGETPLATFORM" = "linux/arm/v7" ] ; then sed -ie s/x86_64/armhf/g assets/script/funcstore.js ; elif [ "$TARGETPLATFORM" = "linux/arm64" ] ; then sed -ie s/x86_64/arm64/g assets/script/funcstore.js; fi

RUN chown -R app:app ./

USER app

CMD ["./gateway"]

