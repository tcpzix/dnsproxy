# sniproxy

Proxies incoming HTTP and TLS connections based on the hostname that is parsed
from either HTTP Host header (for plain HTTP connections) or TLS ClientHello.

This allows transparent proxying of the network traffic by simply rerouting
connections to the `sniproxy`. There are many ways to re-route the traffic, but
most often it is done either on the DNS level or by using `iptables`.

## Features

* Embedded DNS server that can be used to redirect traffic to the proxy.
* Supports both TLS and plain HTTP.
* Supports forwarding connections to an upstream SOCKS proxy.
* Flexible rules for redirecting, forwarding and blocking connections.
* Cross-platform and simple.

## How to install

* Using homebrew:
    ```
    brew install ameshkov/tap/sniproxy
    ```
* From source:
    ```
    go install github.com/ameshkov/sniproxy
    ```
* You can get a binary for your platform from
  the [releases page](https://github.com/ameshkov/sniproxy/releases).

## Using `sniproxy`

### Redirect all traffic to a SNI proxy

* Run `sniproxy` and rewrite DNS responses to point to `1.2.3.4`:
  ```shell
  sudo ./sniproxy --dns-redirect-ipv4-to=1.2.3.4
  ```
* You can test locally it with the following commands:
  ```shell
  curl "https://example.org/" --dns-servers 127.0.0.1
  curl "http://example.org/" --dns-servers 127.0.0.1
  ```

  Not every curl version supports `--dns-servers`. Alternatively, use these
  command:
  ```shell
  curl "https://example.org/" --connect-to example.org:443:127.0.0.1:443
  curl "http://example.org/" --connect-to example.org:80:127.0.0.1:80
  ```

* Now you should just point your device to the DNS server that is running on
  your computer.

### Forward all traffic to a SOCKS proxy

Run `sniproxy`, rewrite DNS responses to point to `1.2.3.4`, :

```shell
sudo ./sniproxy \
    --dns-redirect-ipv4-to=1.2.3.4 \
    --forward-proxy="socks5://127.0.0.1:1080"
```

Now every connection will be re-routed to the SOCKS5 proxy on `127.0.0.1:1080`.

You can limit which domains are re-routed. For instance, here only `example.org`
and `example.com` will be re-routed through the SOCKS5 proxy:

```shell
sudo ./sniproxy \
    --dns-redirect-ipv4-to=1.2.3.4 \
    --forward-proxy="socks5://127.0.0.1:1080" \
    --forward-rule=example.org \
    --forward-rule=example.com
```

### Block domains

You may want to block access to some domains. Here's how to do it:

```shell
sudo ./sniproxy \
    --dns-redirect-ipv4-to=1.2.3.4 \
    --forward-rule=example.org \
    --forward-rule=example.com
```

### Command-line arguments

```shell
Usage:
  sniproxy [OPTIONS]

Application Options:
      --dns-address=          IP address that the DNS proxy server will be listening to. (default: 0.0.0.0)
      --dns-port=             Port the DNS proxy server will be listening to. (default: 53)
      --dns-upstream=         The address of the DNS server the proxy will forward queries that are not rewritten by sniproxy.
                              (default: 8.8.8.8)
      --dns-redirect-ipv4-to= IPv4 address that will be used for redirecting type A DNS queries.
      --dns-redirect-ipv6-to= IPv6 address that will be used for redirecting type AAAA DNS queries.
      --dns-redirect-rule=    Wildcard that defines which domains should be redirected to the SNI proxy. Can be specified multiple
                              times. (default: *)
      --http-address=         IP address the SNI proxy server will be listening for plain HTTP connections. (default: 0.0.0.0)
      --http-port=            Port the SNI proxy server will be listening for plain HTTP connections. (default: 80)
      --tls-address=          IP address the SNI proxy server will be listening for TLS connections. (default: 0.0.0.0)
      --tls-port=             Port the SNI proxy server will be listening for TLS connections. (default: 443)
      --forward-proxy=        Address of a SOCKS proxy that the connections will be forwarded to according to forward-rule.
      --forward-rule=         Wildcard that defines what connections will be forwarded to forward-proxy. Can be specified multiple
                              times. If no rules are specified, all connections will be forwarded to the proxy.
      --block-rule=           Wildcard that defines what domains should be blocked. Can be specified multiple times.
      --verbose               Verbose output (optional)
      --output=               Path to the log file. If not set, write to stdout.

Help Options:
  -h, --help                  Show this help message
```

## Debugging locally

If you want to contribute to `sniproxy`, here are some tips how to debug it
locally.

First, you rarely want to run with `sudo` and instead you'd prefer to use high
ports. `sniproxy` provides command-line arguments for that.

```shell
./sniproxy \
    --dns-address=127.0.0.1 \
    --dns-port=5354 \
    --dns-upstream=8.8.8.8 \
    --dns-redirect-ipv4-to=127.0.0.1 \
    --dns-redirect-rule=example.org \
    --dns-redirect-rule=example.com \
    --tls-address=127.0.0.1 \
    --tls-port=8443 \
    --http-address=127.0.0.1 \
    --http-port=8080 \
    --forward-proxy="socks5://127.0.0.1:1080" \
    --verbose
```

It is easy to use curl to debug `sniproxy`:

```shell
# HTTPS request
curl "https://example.org/" --connect-to example.org:443:127.0.0.1:8443

# Plain HTTP request
curl "http://example.org/" --connect-to example.org:80:127.0.0.1:8080

```

Use `mitmproxy` to debug `sniproxy` with forwarding rules:

```shell
# Run mitmproxy in SOCKS mode
mitmweb --mode=socks5

# Forward connections to mitmproxy
./sniproxy \
    --dns-address=127.0.0.1 \
    --dns-port=5354 \
    --dns-upstream=8.8.8.8 \
    --dns-redirect-ipv4-to=127.0.0.1 \
    --dns-redirect-rule=example.org \
    --dns-redirect-rule=example.com \
    --tls-address=127.0.0.1 \
    --tls-port=8443 \
    --http-address=127.0.0.1 \
    --http-port=8080 \
    --forward-proxy="socks5://127.0.0.1:1080" \
    --verbose

# Check that the connections were properly forwarded
curl "https://example.org/" --connect-to example.org:443:127.0.0.1:8443 --insecure
curl "http://example.org/" --connect-to example.org:80:127.0.0.1:8080

```
