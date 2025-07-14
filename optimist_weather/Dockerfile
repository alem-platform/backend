## Build
FROM golang:1.23.2-alpine3.19 AS buildenv

ADD go.mod go.sum /

RUN go mod download

WORKDIR /app

ADD . .

RUN  go build -o main cmd/main.go

## Deploy
FROM alpine

WORKDIR /

COPY --from=buildenv  /app/ /

EXPOSE 8081

CMD ["/main"]