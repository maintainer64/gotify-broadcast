# Stage 0. Download projects
FROM golang:1.21-alpine as git
RUN apk add build-base git
WORKDIR /app
RUN git clone https://github.com/gotify/server.git --branch v2.4.0 --single-branch /app/server
RUN git clone https://github.com/maintainer64/gotify-broadcast.git --branch v0.4.0 --single-branch /app/broadcast

# Stage 1. Build UI
FROM node:16-alpine AS front-builder
WORKDIR /app
COPY --from=git /app/server/ui /app
RUN yarn install
RUN GENERATE_SOURCEMAP=false yarn build

# Stage 2. Build server & plugins
FROM golang:1.21-alpine as builder
RUN apk add build-base git
RUN GO111MODULE=off go get -u github.com/gotify/plugin-api/cmd/gomod-cap
WORKDIR /app/server
COPY --from=git /app/server /app/server
COPY --from=front-builder /app/build /app/server/ui/build
ENV GONOSUMDB=*
ENV GO111MODULE=on
RUN go mod download
ENV LD_FLAGS="-w -s -X main.Version=2.4.0 -X main.BuildDate= -X main.Commit= -X main.Mode=prod"
RUN GO111MODULE=on CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -ldflags="$LD_FLAGS" -a -installsuffix cgo -o gotify-app
# Build plugins
WORKDIR /app/broadcast
COPY --from=git /app/broadcast /app/broadcast
RUN gomod-cap -from /app/server/go.mod -to go.mod
RUN cp internal/ensure/ensure.go.v2 internal/ensure/ensure.go
RUN go mod tidy
RUN go mod download
RUN go build -mod=readonly -a -installsuffix cgo -ldflags="$LD_FLAGS" -buildmode=plugin -o broadcasts.so

# Stage 3. Final to deploy
FROM alpine:3.18.4 AS deploy
WORKDIR /app
# Copy binary
ENV GIN_MODE=release
COPY --from=builder /app/server/gotify-app /app/gotify-app
COPY --from=builder /app/broadcast/broadcasts.so /app/data/plugins/broadcasts.so
# Open port
EXPOSE 80
ENTRYPOINT ["./gotify-app"]
