FROM alpine:3.17
RUN adduser -s /bin/false -H -D pdns
RUN apk add pdns-recursor \
  && mkdir -p /var/run/pdns-recursor \
  && chown pdns:pdns /var/run/pdns-recursor
USER pdns
ENTRYPOINT ["pdns_recursor", "--daemon=no", "--setuid=pdns", "--setgid=pdns"]
