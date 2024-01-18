FROM certbot/dns-cloudflare:latest

ADD ./scripts /scripts
RUN chmod +x /scripts/cron && chmod +x /scripts/renew

VOLUME /ipc

CMD ["/scripts/cron", "&&", "crond", "-f"]
