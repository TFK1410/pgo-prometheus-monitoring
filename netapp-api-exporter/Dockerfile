FROM golang:1.13.4-alpine as build

ENV APP_NAME="netapp-api-exporter" \
    APP_PATH="/app" \
    APP_BUILD_NAME="netapp-api-exporter" \
    CGO_ENABLED=0 \
    GOOS=linux 

RUN mkdir ${APP_PATH}
WORKDIR ${APP_PATH}

RUN apk add --no-cache git mercurial \
    && git clone https://github.com/sapcc/netapp-api-exporter . \
    && go mod download \ 
    && apk del git mercurial \
    && go build -ldflags="-s -w" -o ${APP_BUILD_NAME} . \
    && chmod +x ${APP_BUILD_NAME}

FROM alpine:3.11

ENV APP_BUILD_PATH="/app" \
    APP_BUILD_NAME="netapp-api-exporter"

WORKDIR ${APP_BUILD_PATH}
COPY --from=build ${APP_BUILD_PATH}/${APP_BUILD_NAME} ${APP_BUILD_PATH}/

EXPOSE 9108
ENTRYPOINT [ "./netapp-api-exporter" ]