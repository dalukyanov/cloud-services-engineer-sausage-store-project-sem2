# Stage 1: build
FROM golang:1.22-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Stage 2: runtime
FROM alpine:3.20

RUN apk --no-cache add ca-certificates dumb-init \
    && adduser -D -u 1000 appuser

WORKDIR /app

COPY --from=builder /app/main ./main

USER appuser

EXPOSE 8080

ENTRYPOINT ["dumb-init", "--"]
CMD ["./main"]