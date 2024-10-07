Running Pynt with Custom CA
---

## Testing Goat

1. Run the proxy via Docker Compose, it will be available on `https://localhost:8443`
    ```sh
    docker compose up -d
    ```

1. Run the tests
    ```sh
    go test .
    ```

## Development

### Generating a Custom CA

```sh
# Generate the CA key and certificate
openssl genrsa -out certs/root.key 4096
openssl req -x509 -new -nodes -key certs/root.key -sha256 -days 3650 -out certs/root.pem -subj "/C=US/ST=CA/O=Proxy, Inc./CN=Custom CA"

# Generate a key and a certificate request for `localhost`
openssl genrsa -out certs/localhost.key 2048
openssl req -new -sha256 -config certs/localhost.cnf -key certs/localhost.key -subj "/C=US/ST=CA/O=MyOrg, Inc./CN=localhost" -out certs/localhost.csr

# Sign the CSR using the CA
openssl x509 -req -in certs/localhost.csr -CA certs/root.crt -CAkey certs/root.key -CAcreateserial -out certs/localhost.crt -extfile certs/localhost.cnf -extensions v3_ext -days 365 -sha256

# Create a combined cert
cat certs/localhost.crt certs/root.crt > certs/localhost-with-ca.crt
```
