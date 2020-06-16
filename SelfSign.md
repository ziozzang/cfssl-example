
# CertGen

## Install cfssl

```
mkdir -p /opt/bin/ ; \
wget https://github.com/cloudflare/cfssl/releases/download/v1.4.1/cfssl_1.4.1_linux_amd64 -O /opt/bin/cfssl ; \
wget https://github.com/cloudflare/cfssl/releases/download/v1.4.1/cfssljson_1.4.1_linux_amd64 -O /opt/bin/cfssljson ; 
chmod +x /opt/bin/cf*
```

## Create Root cert

```
mkdir -p /opt/certgen
cd /opt/certgen

cat > /opt/certgen/root_ca.cfg <<EOF
{
    "CN": "Some Root CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C":  "KR",
            "ST": "Seoul",
            "L":  "Somewhere",
            "O":  "Personal",
            "OU": "Home"
        }
    ],
    "ca": {
    	"expiry": "262800h"
 		}
}
EOF
/opt/bin/cfssl genkey -loglevel=0 -initca /opt/certgen/root_ca.cfg | /opt/bin/cfssljson -bare /opt/certgen/root_ca

```

## Create Intermediate cert

```

cat > /opt/certgen/int_ca.cfg <<EOF
{
    "CN": "Some IntermediateCA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C":  "KR",
            "ST": "Seoul",
            "L":  "Somewhere",
            "O":  "Personal",
            "OU": "Home"
        }
    ],
    "ca": {
    	"expiry": "87600h"
 		}
}
EOF
/opt/bin/cfssl gencert -initca /opt/certgen/int_ca.cfg | /opt/bin/cfssljson -bare int_ca

# 인터미디엇 CA를 루트 CA로 서명 처리
cat > /opt/certgen/int_ca_sign.cfg <<EOF
{
   "signing": {
       "default": {
           "usages": ["digital signature","cert sign","crl sign","signing"],
           "expiry": "262800h",
           "ca_constraint": {
               "is_ca": true,
               "max_path_len":0,
               "max_path_len_zero": true
           }
       }
   }
}
EOF
/opt/bin/cfssl sign -ca root_ca.pem -ca-key root_ca-key.pem \
  -config /opt/certgen/int_ca_sign.cfg \
  int_ca.csr | /opt/bin/cfssljson -bare int_ca
```

## Check Certs

```
# 정보 보기
openssl req -noout -text -in root_ca.csr
openssl req -noout -text -in int_ca.csr

# Just Copy (Same format)
cp root_ca.pem root_ca.crt

```

## Install Root CA

### Ubuntu

```

mkdir -p /usr/local/share/ca-certificates/extra
cp root_ca.pem /usr/local/share/ca-certificates/extra/hcc_root_ca.crt
update-ca-certificates

```


## Create SAN cert

```

SAN="10.23.45.18"

cat > host_${SAN}.cfg <<EOF
{
   "CN": "${SAN}",
   "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C":  "KR",
            "ST": "Seoul",
            "L":  "Somewhere",
            "O":  "Personal",
            "OU": "Home"
        }
    ],
    "Hosts": [
      "${SAN}",
      "*.${SAN}"
    ],
     "default": {
       "expiry": "8760h"
     }
 }
 
EOF
 
 # 도메인 인증서를 인터미디엇으로 싸이닝
 cat > host_${SAN}_sign.cfg << EOF
 {
   "signing": {
     "profiles": {
       "CA": {
         "usages": ["cert sign"],
         "expiry": "8760h"
       }
     },
     "default": {
       "usages": ["digital signature"],
       "expiry": "8760h"
     }
   }
}
EOF

/opt/bin/cfssl  gencert -loglevel=0 \
  -ca int_ca.pem -ca-key int_ca-key.pem \
  -config host_${SAN}_sign.cfg \
  host_${SAN}.cfg | /opt/bin/cfssljson -bare host_${SAN}

openssl req -noout -text -in host_${SAN}.csr

# Server Certs
cat host_${SAN}.pem int_ca.pem > host_${SAN}-export.crt
# Server Keys
cat host_${SAN}-key.pem

```
