Running Pynt with a Custom CA
---

## Testing Goat

1. Configure your machine to point `proxy.goat.internal` to `127.0.0.1` by adding the following to `/etc/hosts`:
    ```txt
    127.0.0.1   proxy.goat.internal
    ```

1. Run the proxy via Docker Compose, it will be available on `https://proxy.goat.internal:8443`
    ```sh
    docker compose up -d
    ```

1. Run the tests
    ```sh
    go test -v .
    ```

### Why can't I use `127.0.0.1` or `localhost` as a target in Go?

Go's default proxy implementation ([`http.ProxyFromEnvironment`](https://pkg.go.dev/net/http#ProxyFromEnvironment)) does **not** use proxy for requests to [`localhost`](https://github.com/golang/net/blob/6cc5ac4e9a03d73b331eb1d6db98a02e558243b7/http/httpproxy/proxy.go#L177-L179) or [`127.0.0.1`](https://github.com/golang/net/blob/6cc5ac4e9a03d73b331eb1d6db98a02e558243b7/http/httpproxy/proxy.go#L180-L185).

If running on a remote machine with it's own IP/DNS - we don't have any issue.

When we are running these tests, we assume the target is at `proxy.goat.internal`, and we issue a certificate for this domain accordingly.

## Development

### Generating a Custom CA

```sh
# Generate the CA key and certificate
openssl genrsa -out certs/root.key 4096
openssl req -x509 -new -nodes -key certs/root.key -sha256 -days 3650 -out certs/root.pem -subj "/C=US/ST=CA/O=Proxy, Inc./CN=Custom CA"

# Generate a key and a certificate request for `proxy.goat.internal`
openssl genrsa -out certs/proxy-goat-internal.key 2048
openssl req -new -sha256 -config certs/proxy-goat-internal.cnf -key certs/proxy-goat-internal.key -out certs/proxy-goat-internal.csr

# Sign the CSR using the CA
openssl x509 -req -in certs/proxy-goat-internal.csr -CA certs/root.crt -CAkey certs/root.key -CAcreateserial -out certs/proxy-goat-internal.crt -extfile certs/proxy-goat-internal.cnf -extensions v3_ext -days 365 -sha256

# Create a combined cert
cat certs/proxy-goat-internal.crt certs/root.crt > certs/proxy-goat-internal-bundle.crt
```

### Using a Custom CA for HTTP(s) calls in in Go

```golang
rootCAs := x509.NewCertPool()
cert, err := os.ReadFile("path/to/root-ca")
if err != nil {
    return nil, fmt.Errorf("unable to load root ca: %w", err)
}
if ok := rootCAs.AppendCertsFromPEM(cert); !ok {
    return nil, fmt.Errorf("unable to create a root CA list")
}

tls_config := &tls.Config{
    InsecureSkipVerify: false,
    RootCAs:            rootCAs,
}
transport := &http.Transport{
    TLSClientConfig: tls_config,
    Proxy:           http.ProxyFromEnvironment, // Same as `http.DefaultTransport`
}

client := &http.Client{Transport: transport}
```
