FROM ubuntu:20.04
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y golang-go
COPY app.go .
RUN CGO_ENABLED=0 go build app.go

FROM alpine:3.14.2
RUN addgroup -S group && adduser -S lameuser -G group -h /home/lameuser
COPY --from=0 /app /home/lameuser
RUN chmod u-w /etc && rm /bin/*
USER lameuser
CMD ["/home/lameuser/app"]
