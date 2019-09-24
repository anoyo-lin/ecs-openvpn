FROM alpine:latest
RUN (apk -Uuv add --no-cache openvpn iptables ca-certificates && \
  rm -f /var/cache/apk/*)
ONBUILD COPY openvpn /etc/openvpn
ONBUILD WORKDIR /etc/openvpn
ONBUILD ENTRYPOINT ["./openvpn-start.sh"]
