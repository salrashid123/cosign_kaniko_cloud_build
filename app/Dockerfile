# go1.19.8 linux/amd64
FROM docker.io/golang@sha256:9f2dd04486e84eec72d945b077d568976981d9afed8b4e2aeb08f7ab739292b3 as build 

WORKDIR /go/src/app
COPY . .
RUN go mod download
RUN GOOS=linux GOARCH=amd64 go build -buildvcs=false -o /go/bin/server

RUN groupadd -g 65532 nonroot && \
  useradd nonroot -u 65532 -g 65532 -m -s /sbin/nologin && \
  chown nonroot:nonroot /go/bin/server && \
  chown -R nonroot:nonroot /go/src/app/

# base-debian11-nonroot
FROM gcr.io/distroless/base-debian11@sha256:10985f0577b62767a2f0218bff7dec33c59a5ef9a587d889bf0861e0c3123d57

COPY --from=build /go/bin/server /home/nonroot/server
COPY --from=build /go/src/app/config.json /home/nonroot/config.json
EXPOSE 8080
ENTRYPOINT ["./server"]
CMD []