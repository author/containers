FROM alpine:latest
RUN apk update \
  && apk add git jq curl \
  && mkdir -p /app

COPY setup /app/
RUN chmod +x /app/setup

VOLUME /output

ENV USERNAME=""
ENV TOKEN=""

WORKDIR /app

CMD ["/bin/sh", "/app/setup"]