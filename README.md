| Supported Targets | ESP32-C3 | ESP32-C6 | ESP32-H2 | ESP32-S2 | ESP32-S3 |
| ----------------- | -------- | -------- | -------- | -------- | -------- |

# ESP_HTTP Mutual Authentication with Digital Signature

Espressif's ESP32-S2, ESP32-S3, ESP32-C3, ESP32-C6 and ESP32-H2 MCU have a built-in Digital Signature (DS) Peripheral, which provides hardware acceleration for RSA signature. More details can be found at [Digital Signature with ESP-TLS](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s2/api-reference/protocols/esp_tls.html#digital-signature-with-esp-tls).


## How to use example

### Hardware Required

This example can be executed on any ESP32-S2, ESP32-S3, ESP32-C3 or ESP32-C6 board (which has a built-in DS peripheral), the only required interface is WiFi and connection to internet.

### Configure the project

#### 1) Patching esp-idf
```
    cd $ESP_IDF
    patch -p0 -i esp_http_ds_peripheral.patch

```

#### 2) Selecting the target
As the project is to be built for the target ESP32-S2, ESP32-S3, ESP32-C3, ESP32-C6 or ESP32-H2 it should be selected with the following command
```
idf.py set-target /* target */
```
More detials can be found at [Selecting the target](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/build-system.html#selecting-the-target).

#### 3) Generate your client key and certificate


1) Generate root ca and key pair:
```
openssl req -newkey rsa:2048 -nodes -keyout prvtkey.pem -x509 -days 3650 -out cacert.pem -subj "/CN=Test CA"
```

2) Generate client private key:
```
openssl genrsa -out client.key
```

3) Generate device cert:
```
openssl req -out client.csr -key client.key -new
openssl x509 -req -days 365 -in client.csr -CA cacert.pem -CAkey prvtkey.pem  -sha256 -CAcreateserial -out client.crt
```


#### 4) Configure the DS peripheral

* i) Install the [esp_secure_cert configuration utility](https://github.com/espressif/esp_secure_cert_mgr/tree/main/tools#esp_secure_cert-configuration-tool) with following command:
```
pip install esp-secure-cert-tool
```
* ii) The DS peripheral can be configured by executing the following command:

```
configure_esp_secure_cert.py -p /dev/ttyUSB-4.2-00 --keep_ds_data_on_host --efuse_key_id 1 --ca-cert cacert.pem --device-cert client.crt --private-key client.key --target_chip esp32s3 --secure_cert_type cust_flash_tlv --configure_ds --priv_key_algo RSA 2048 --skip_flash
```
This command shall generate a partition named `esp_secure_cert.bin` in the `esp_secure_cert_data` directory. This partition would be aumatically detected by the build system and flashed at appropriate offset when `idf.py flash` command is used. For this process, the command must be executed in the current folder only.

In the command USB COM port is nothing but the serial port to which the ESP chip is connected. see
[check serial port](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/establish-serial-connection.html#check-port-on-windows) for more details.
RSA private key is nothing but the client private key ( RSA ) generated in Step 2.

> Note: More details about the `esp-secure-cert-tool` utility can be found [here](https://github.com/espressif/esp_secure_cert_mgr/tree/main/tools).

#### 5) Connection cofiguration
* Open the project configuration menu (`idf.py menuconfig`)
* Configure Wi-Fi or Ethernet under "Example Connection Configuration" menu. See "Establishing Wi-Fi or Ethernet Connection" section in [examples/protocols/README.md](../../README.md) for more details.

### Build and Flash

Build the project and flash it to the board, then run monitor tool to view serial output:

```
idf.py -p PORT flash monitor
```

(To exit the serial monitor, type ``Ctrl-]``.)

See the Getting Started Guide for full steps to configure and use ESP-IDF to build projects.

### Running example server

```
npm install express
openssl req -x509 -newkey rsa:4096 -keyout server_key.pem -out server_cert.pem -nodes -days 365 -subj "/CN=localhost/O=Client\ Certificate\ Demo"
env NODE_DEBUG='tls,https' DEBUG=express:*  node server.js

```


### Test server using curl

```
openssl pkcs12 -export -in client.crt -inkey client.key -out client.p12
curl -v --insecure  https://192.168.1.6:9999/authenticate --cert client.p12 --cert-type p12
```

## Example Output

```
I (4180) wifi:set rx beacon pti, rx_bcn_pti: 0, bcn_timeout: 25000, mt_pti: 0, mt_time: 10000
I (4270) wifi:AP's beacon interval = 102400 us, DTIM period = 1
I (5190) esp_netif_handlers: example_netif_sta ip: 192.168.2.181, mask: 255.255.248.0, gw: 192.168.1.1
I (5190) example_connect: Got IPv4 event: Interface "example_netif_sta" address: 192.168.2.181
I (5480) example_connect: Got IPv6 event: Interface "example_netif_sta" address: fe80:0000:0000:0000:7edf:a1ff:fef3:d368, type: ESP_IP6_ADDR_IS_LINK_LOCAL
I (5480) example_common: Connected to example_netif_sta
I (5490) example_common: - IPv4 address: 192.168.2.181,
I (5490) wifi:<ba-add>idx:1 (ifx:0, 68:d7:9a:45:c5:22), tid:0, ssn:732, winSize:64
I (5500) example_common: - IPv6 address: fe80:0000:0000:0000:7edf:a1ff:fef3:d368, type: ESP_IP6_ADDR_IS_LINK_LOCAL
I (5510) DS_HTTPS_EXAMPLE: Server cert: '-----BEGIN CERTIFICATE-----
MIIFTTCCAzWgAwIBAgIUGIRoB/XSmta2QvEajgMcsdiw2C8wDQYJKoZIhvcNAQEL
BQAwNjESMBAGA1UEAwwJbG9jYWxob3N0MSAwHgYDVQQKDBdDbGllbnQgQ2VydGlm
aWNhdGUgRGVtbzAeFw0yMzEwMTgyMDAxMzlaFw0yNDEwMTcyMDAxMzlaMDYxEjAQ
BgNVBAMMCWxvY2FsaG9zdDEgMB4GA1UECgwXQ2xpZW50IENlcnRpZmljYXRlIERl
bW8wggIiMA0GCSqGSIb3DQEBAQUAA4ICDwAwggIKAoICAQCI8I1MznV0yRwiss9B
IwjK2EfBp0DW1iYcEN2TgK1qScc/d2Unjdt5crvlUY2r/HyCRnXWVSPeW3cLr2gN
oilwHsKKMBZ96anKp6p9PA8Ccr7dL2Z1YaYggc2wm2hZ9YSCuGhEun6a/ZOfk5ft
dSlYgQynQah/oGf4sTos8huncXa5iq5VYjMMP48IuyQDg8P3KagCGJmYrNz8JWM8
HMOA1qIINwveF87vSjYmPrvKWjAmCOaNuUDlki+4NW3yvxWOhUykWke5ldWrwcaT
+7+OiwqjlULWhdc7FWsyKMmFyjupe0ZJ1CFETnpCeLeo7LqUSRTOWMq04xOG7eca
DPFXzGmdhoaBZaDzSHtSeE6bzPXJhuCNE1n/j6dMBMWjdp/McQahDxdNxFrB2CrQ
fI+4URLkA9iSPwgJAS3JjG/89hNzec2QAjhq98kF1kb4OxlwyLjvQg4qaL23gMnT
6XB8u3tSeSXSuTVnMZTDsgCTVwdaHSewkYI3w3YtF4DIWJdplR/OBEQL/8CENKkT
IfK/Njt3OuMALMExWLpylo1n9wt4l/fu3aVlD5KHlfshu98Kn6HMvIs9KaCz6R2O
aMqRg3R2h1Qr9jN6E21YXAWZ9+Kcpox8F3qzEexOkThYge/mRTxfwaYA14kHRWwg
GG937AjmMjOgywoukFZN0nKZvQIDAQABo1MwUTAdBgNVHQ4EFgQUB5wnORvpqHRZ
TtD8Tw+/gxNdW/MwHwYDVR0jBBgwFoAUB5wnORvpqHRZTtD8Tw+/gxNdW/MwDwYD
VR0TAQH/BAUwAwEB/zANBgkqhkiG9w0BAQsFAAOCAgEAYp97g98ErGroisbFu+E7
LtHP7r8RqrODT40PTOJN9sX7u/KrmH2OmL61xy6Euz4tuvQQg0kKA+AodqjT3NKD
vhJUR5e+FnRkR3qdboe0QyuJlelPfILkTRyctQCQRIdrtlb2UZCiFjgIcJSGm4fS
DTqkGPay04t8LWjX0YYHiHPi8DJOf/JBxzPPtCiXtdrrsySYQZtmAC3C9XfZdBuK
DahC1yZDWewqyMAg+9T9hVtSMKFkRDM9AbqIxXcq+oQhiuDGxsGqnG/Mgg6jX4IX
nbm/yDBJpXkH00piipuOCjm1vm9RHPcVjsMjzc94O+EmswZbVo81+cboXM6J0ZjI
C3t/+sGmyzfanqkSLo/Kb4AE72ka8kuOd+WsI5TCSo3g4cIWzWk5VmePxwhRu/av
7ZFeVt7CTtcjEP2g+TVXQHIS1euJQz0ZIkWNR+1jhTCN9Tv9rl796XFWr+B5wjAs
b75EHfdOIwneJixldCfRc4QoP3AzpG/ho4/2N05Fgv4gNFoulutWytoe9i/W65B4
FBSFAD2/P5SFNCzlSvMk5UJ7WjInFiishlkTPM2rBFgKV0fESWrnF/NBHyGMZnbk
L1L9biiY7ukdQecySlMiKBMPE+xAM5UEdNDtPRx/SdJp+a4VRJz3eHKohxQ3AbXn
sBlz1fxtJVwO6XnbuU3BwQg=
-----END CERTIFICATE-----
'
I (5680) DS_HTTPS_EXAMPLE: Deivce cert: '-----BEGIN CERTIFICATE-----
MIIC2TCCAcECFFYh3QgGUuHNW72X61a0jjj5faO0MA0GCSqGSIb3DQEBCwUAMBIx
EDAOBgNVBAMMB1Rlc3QgQ0EwHhcNMjMxMDE4MDk0NzIyWhcNMjQxMDE3MDk0NzIy
WjBAMQswCQYDVQQGEwJTRTESMBAGA1UECAwJU3RvY2tob2xtMQ8wDQYDVQQKDAZU
aWJiZXIxDDAKBgNVBAMMA1RCMTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoC
ggEBANnshgztDvRQuAJzyTCAaGlcTStacL7bzurQsY8qNv3UJqaotTXfcsL1CZTO
56CWoYOYd0BifmuPTYWLXxymix6KapkoaWCAfmShiPMqhgerRI0SvTH/2Ma/bEKi
IjPlm0/gYj9tFjx560+VmRDtdLscuv/1JHXsAnpYzoXSPyo3Ep+jwuS5YGEx1Nhi
3ye7B7HwgJY2tr+nICZdHawylH48xMhKlTlt8Km+voc3xey1Tor+cDKdOXVZaJZT
45Y3Lz1sLZgX4xipm4Z1zuBxVzhT5cqOpeGlkFUY+ZV+6tBQN85mnbiyIJA3Yvov
mO7sKUYM02nzsz1kN1diWDKa4k0CAwEAATANBgkqhkiG9w0BAQsFAAOCAQEAlxrG
BPAcXC7q/+cuHlkAzC+BDdLQSYKTJYM+fSEtrm24Q7juDSneWih0pIwK5GX0o9mc
9rsXGavalWek7Dn0ndqB6P2fAiV1HPDG0rgGyBvNJojKhIpawgE4Zf8s7WCHOOH0
iRqrVPRIm5ROwGJtl5isdy5VgWFdCZ0edmQdnKZZUb9AhFGn4mAPWe62HKAXVA7/
fv9jilletMfgxMew82YtOtq3o0ZggxoLopdTk17UB/UvAMZa7yUAiRcHVVMBBRvg
uJ1pF84eBK56e/0NFpYEwh7snRS3KcZs+xIJT0FVBeqptthaa+4Bw7+kjkeUl7IA
c5tH8kW1cCvEDb7xwA==
-----END CERTIFICATE-----
'
W (5790) esp-tls-mbedtls: ds_data: 0x3fcb0120
W (5800) esp-tls-mbedtls: DS peripheral is used for the TLS handshake
Hello TB1, your certificate was issued by Test CA!
I (6430) DS_HTTPS_EXAMPLE: HTTPS Status = 200, content_length = 50
I (6430) DS_HTTPS_EXAMPLE: HTTP_EVENT_DISCONNECTED
I (6440) main_task: Returned from app_main()
```
