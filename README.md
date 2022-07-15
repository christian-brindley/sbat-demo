[ req ]
distinguished_name = server_distinguished_name
x509_extensions = server_extensions

[ server_distinguished_name ]
commonName           = CN
commonName_default   = sbat.demo

[ server_extensions ]
subjectAltName       = @alternate_names

[ alternate_names ]
DNS.1  = tpp.sbat.demo
DNS.2  = rs.sbat.demo
DNS.3  = consent.sbat.demo
DNS.4  = rcs.sbat.demo
DNS.5  = swagger.sbat.demo


openssl req -x509 -newkey rsa:4096 -keyout sbat-demo.key -out sbat-demo.crt -sha256 -days 365 -subj '/CN=sbat.demo' -nodes  -config openssl.cnf 
kubectl create secret tls server.tls --key=sbat-demo.key --cert=sbat-demo.crt
