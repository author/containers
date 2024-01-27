FROM certbot/dns-cloudflare:latest

ADD ./scripts /scripts
RUN chmod +x /scripts/cron && chmod +x /scripts/renew

VOLUME /ipc

ENTRYPOINT ["/scripts/cron"]
