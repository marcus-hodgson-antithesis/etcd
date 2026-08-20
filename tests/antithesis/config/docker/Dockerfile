ARG GO_VERSION=1.25.5

FROM golang:$GO_VERSION AS build
RUN go install github.com/a8m/envsubst/cmd/envsubst@v1.4.3

ARG IMAGE_TAG
COPY docker-composes/docker-compose.yaml /docker-compose.yaml.template
RUN IMAGE_TAG=${IMAGE_TAG} cat /docker-compose.yaml.template | envsubst > /docker-compose.yaml

FROM scratch
COPY --from=build /docker-compose.yaml /docker-compose.yaml
