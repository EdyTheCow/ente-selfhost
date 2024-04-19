FROM golang:1.20-alpine3.17

RUN apk add --no-cache gcc musl-dev git libsodium-dev

WORKDIR /etc/

RUN git clone https://github.com/ente-io/ente.git && mv ente/server /etc/server && rm -r /etc/ente

WORKDIR /etc/server/

RUN go mod download
