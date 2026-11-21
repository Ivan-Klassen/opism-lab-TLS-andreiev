<!--
Бланк звіту до лабораторної роботи «TLS та сертифікати».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ | Андреєв Іван Олегович |
| Група | ІПЗ_2.01 |
| Ідентифікатор | se-201-01 |
| Дата виконання | 15.11.2026 |
| Середовище | Docker на Debian Linux|
| Версія OpenSSL | OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024) |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер | origin.se-201-01.lab |
| Імена для сертифіката | www.se-201-01.lab, news.se-201-01.lab |
| Ваш сервер (IP) | 10.53.1.1 |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```text
root@se-201-01:/# cd /lab/work                  
timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt &
sleep 1; curl -s http://origin.se-201-01.lab/; wait
grep -a -E 'GET /|Host:|server: web' t1-http.txt
[1] 61
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: origin.se-201-01.lab
SNI: 
protocol: HTTP/1.1 
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
16:50:51.680539 IP 10.53.1.1.36908 > 10.53.0.80.80: Flags [P.], seq 1:84, ack 1, win 502, options [nop,nop,TS val 826795498 ecr 316290251], length 83: HTTP: GET / HTTP/1.1
1G....4.GET / HTTP/1.1
Host: origin.se-201-01.lab
server: web
Host: origin.se-201-01.lab
```

Вивід кроку 1.2:

```text
root@se-201-01:/lab/work# timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt &
sleep 1; curl -s --cacert $STAND_CA https://origin.se-201-01.lab/; wait
grep -a -c -E 'GET /|server: web' t1-https.txt
grep -a -o 'origin.se-201-01.lab' t1-https.txt | wc -l
[1] 67
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: origin.se-201-01.lab
SNI: origin.se-201-01.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
0
1
```

| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` | так | ні |
| заголовок `Host` | так | так |
| тіло відповіді (`server: web`) | так | так |
| ім'я віддаленого сервера (кількість входжень) | 0 | 1 |

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?
Host: origin.se-201-01.lab

2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?
Для HTTP немає рукостискання, тому не передається радок "SNI:".

3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?
Бо сервер обирає протокол ALPN, тому ми отримали "HTTP/2.0" в HTTPS.

## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```text
root@se-201-01:/# openssl s_copenssl s_client -connect origin.se-201-01.lab:443 -servername origin.se-201-01.lab -alpn h2,http/1.1 \
  -CAfile $STAND_CA -trace </dev/null > t2-13.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-13.txt
grep -a -E '^(ALPN|New,)|Verify return' t2-13.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=331
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Certificate, Length=989
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=75
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Alert (21)
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
ALPN protocol: h2
Verify return code: 0 (ok)

```

Вивід кроку 2.2 (TLS 1.2):

```text
root@se-201-01:/# openssl s_copenssl s_client -connect origin.se-201-01.lab:443 -servername origin.se-201-01.lab -alpn h2,http/1.1 \
  -tls1_2 -CAfile $STAND_CA -trace </dev/null > t2-12.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-12.txt
grep -a -E '^(ALPN|New,)|Verify return' t2-12.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=226
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=984
Received Record
  Content Type = Handshake (22)
    ServerKeyExchange, Length=110
Received Record
  Content Type = Handshake (22)
    ServerHelloDone, Length=0
Sent Record
  Content Type = Handshake (22)
    ClientKeyExchange, Length=33
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = Handshake (22)
    NewSessionTicket, Length=230
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = ApplicationData (23)
Sent Record
  Content Type = Alert (21)
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES256-GCM-SHA384
ALPN protocol: h2
    Verify return code: 0 (ok)

```

<!-- Один рядок на кожен запис з виводу 2.1, у тому самому порядку. Напрям: К→С або С→К. -->

| № | Напрям | `Content Type` | `Inner Content Type` | Повідомлення | `Length` |
|---|---|---|---|---|---|
| 1 | К→С | Handshake (22) | — | ClientHello | 331 |
| 2 | С→К | Handshake (22) | ServerHello | — | 118 |
| 3 | С→К | ChangeCipherSpec (20) | — | — | — |
| 4 | С→К | ApplicationData (23) | Handshake (22) | EncryptedExtensions | 15 |
| 5 | С→К | ApplicationData (23) | Handshake (22) | Certificate | 989 |
| 6 | С→К | ApplicationData (23) | Handshake (22) | CertificateVerify | 75 |
| 7 | С→К | ApplicationData (23) | Handshake (22) | Finished | 48 |
| 8 | К→С | ChangeCipherSpec (20) | — | — | — |
| 9 | К→С | ApplicationData (23) | Handshake (22) | Finished | 48 |
| 10 | К→С | ApplicationData (23) | Alert (21) | — | — |

Вивід кроків 2.3 і 2.4:

```text
root32mroot@se-201-01:/# grep -a -A3 'extension_type=server_name' t2-13.txt | head -4

for v in 1.3 1.2; do
  timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt &
  sleep 1; curl -s -o /dev/null --tlsv1.2 --tls-max $v --cacert $STAND_CA https://origin.se-201-01.lab/; wait
  echo "TLS $v: $(grep -a -c 'OPISM Stand Intermediate' t2-dump-$v.txt)"
done
        extension_type=server_name(0), length=25
          0000 - 00 17 00 00 14 6f 72 69-67 69 6e 2e 73 65 2d   .....origin.se-
          000f - 32 30 31 2d 30 31 2e 6c-61 62                  201-01.lab
        extension_type=ec_point_formats(11), length=4
[1] 109
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.3: 0
[1] 115
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.2: 2
```

| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт | 331 | 226 |
| Номер першого запису, тип якого спостерігач не бачить | 4 | відсутній |
| Кількість RTT від ClientHello до першого HTTP-запиту | 1 | 2 |
| Чи видно сертифікат у дампі (результат 2.4) | ні (0) | так (2) |
| Обраний протокол ALPN | h2 | h2 |

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?
Шифрування починається з 4-го запису (EncryptedExtensions). Він бачить прикладні дані: ApplicationData (23) та розмір даних.

2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.
Це розширення key_share. Клієнт у TLS 1.3 одразу надсилає свій відкритий ключ.

3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?
Це 5 службових байтів (заголовок списку імен, тип хоста та довжина імені) + довжина доменного імені origin.se-201-01.lab. Доменне ім'я origin.se-201-01.lab важить 20 байтів, тому взагалом буде 25 байтів. 

4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?
У TLS 1.3 запис ChangeCipherSpec передається для сумісності зі старим проміжним обладнанням, яке очікує TLS 1.2.

## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```text
root@se-201-01:/# acme-root
openssl x509 -in /lab/tls/acme-root.crt -noout -subject -dates
curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-01.lab/
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 3b9acc
notBefore=Nov 20 07:16:17 2026 GMT
notAfter=Nov 20 07:16:17 2056 GMT
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text
root@se-201-01:/# lego --servlego --server $ACME_DIR --email admin@se-201-01.lab --accept-tos --path /lab/work/lego \
  --http --http.webroot /var/www/acme \
  --domains www.se-201-01.lab --domains news.se-201-01.lab run --no-bundle
grep acme-challenge /var/log/nginx/access.log
2026/11/20 18:27:42 [INFO] [www.se-201-01.lab, news.se-201-01.lab] acme: Obtaining SAN certificate
2026/11/20 18:27:42 [INFO] [www.se-201-01.lab] AuthURL: https://ca.lab:14000/authZ/v3U7zsn5v5AVtgsoggKIH8XynyjpuCepVbWKjH4DctU
2026/11/20 18:27:42 [INFO] [news.se-201-01.lab] AuthURL: https://ca.lab:14000/authZ/BOrfVY2wLc8qwBPIijAd97AMRH72EJPXlnFxDzoylBI
2026/11/20 18:27:42 [INFO] [www.se-201-01.lab] acme: Could not find solver for: tls-alpn-01
2026/11/20 18:27:42 [INFO] [www.se-201-01.lab] acme: use http-01 solver
2026/11/20 18:27:42 [INFO] [news.se-201-01.lab] acme: authorization already valid; skipping challenge
2026/11/20 18:27:42 [INFO] [www.se-201-01.lab] acme: Trying to solve HTTP-01
2026/11/20 18:27:45 [INFO] [www.se-201-01.lab] The server validated our request
2026/11/20 18:27:45 [INFO] [www.se-201-01.lab, news.se-201-01.lab] acme: Validations succeeded; requesting certificates
2026/11/20 18:27:45 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/20 18:27:45 [INFO] [www.se-201-01.lab] Server responded with a certificate.
10.53.0.10 - - [20/Nov/2026:18:27:21 +0000] "GET /.well-known/acme-challenge/ofAM8HdyBSXzyr7MxCH_1ZxaEh3enojJZ-1YG8I-0KQ HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [20/Nov/2026:18:27:21 +0000] "GET /.well-known/acme-challenge/ofAM8HdyBSXzyr7MxCH_1ZxaEh3enojJZ-1YG8I-0KQ HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [20/Nov/2026:18:27:21 +0000] "GET /.well-known/acme-challenge/ofAM8HdyBSXzyr7MxCH_1ZxaEh3enojJZ-1YG8I-0KQ HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [20/Nov/2026:18:27:24 +0000] "GET /.well-known/acme-challenge/LQOeyqR_7kVFZlRbEMMyZqDXFnqrM_jQdgEHEAhT258 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [20/Nov/2026:18:27:24 +0000] "GET /.well-known/acme-challenge/LQOeyqR_7kVFZlRbEMMyZqDXFnqrM_jQdgEHEAhT258 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [20/Nov/2026:18:27:24 +0000] "GET /.well-known/acme-challenge/LQOeyqR_7kVFZlRbEMMyZqDXFnqrM_jQdgEHEAhT258 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [20/Nov/2026:18:27:42 +0000] "GET /.well-known/acme-challenge/lOscjEkVeTYvTaED7LGXEQsSrBkXBKRwPi01wwm08NA HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [20/Nov/2026:18:27:42 +0000] "GET /.well-known/acme-challenge/lOscjEkVeTYvTaED7LGXEQsSrBkXBKRwPi01wwm08NA HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [20/Nov/2026:18:27:42 +0000] "GET /.well-known/acme-challenge/lOscjEkVeTYvTaED7LGXEQsSrBkXBKRwPi01wwm08NA HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
```

Вивід кроку 3.4:

```text
root@se-201-01:/# cd /lab/work/lego/certificates
openssl x509 -in www.se-201-01.lab.crt -noout -subject -issuer -dates -serial -ext subjectAltName
openssl x509 -in www.se-201-01.lab.issuer.crt -noout -subject -issuer
subject=CN = www.se-201-01.lab
issuer=CN = Pebble Intermediate CA 1e133f
notBefore=Nov 20 18:27:45 2026 GMT
notAfter=Nov 20 18:27:44 2031 GMT
serial=493649775F4B1FA0
X509v3 Subject Alternative Name: 
    DNS:www.se-201-01.lab, DNS:news.se-201-01.lab
subject=CN = Pebble Intermediate CA 1e133f
issuer=CN = Pebble Root CA 3b9acc
```

Вивід кроку 3.5:

```text
root@se-201-01:/lab/work/lego/certificates# cp www.se-2cp www.se-201-01.lab.crt /lab/tls/server.crt
cp www.se-201-01.lab.key /lab/tls/server.key
web-reload
curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-01.lab/
openssl s_client -connect www.se-201-01.lab:443 -servername www.se-201-01.lab -CAfile /lab/tls/acme-root.crt \
  </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/20 18:28:18 [notice] 174#174: signal process started
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
Certificate chain
 0 s:CN = www.se-201-01.lab
   i:CN = Pebble Intermediate CA 1e133f
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 20 18:27:45 2026 GMT; NotAfter: Nov 20 18:27:44 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```

Вивід кроку 3.6:

```text
root@se-201-01:/lab/work/lego/certificates# cat www.se-cat www.se-201-01.lab.crt www.se-201-01.lab.issuer.crt > /lab/tls/server.crt
web-reload
curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-01.lab/
openssl s_client -connect www.se-201-01.lab:443 -servername www.se-201-01.lab -CAfile /lab/tls/acme-root.crt \
  </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
curl -sS --cacert /lab/tls/acme-root.crt https://news.se-201-01.lab/
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/20 18:28:31 [notice] 182#182: signal process started
server: se-201-01 (https)
Host: www.se-201-01.lab
SNI: www.se-201-01.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.se-201-01.lab
   i:CN = Pebble Intermediate CA 1e133f
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 20 18:27:45 2026 GMT; NotAfter: Nov 20 18:27:44 2031 GMT
 1 s:CN = Pebble Intermediate CA 1e133f
   i:CN = Pebble Root CA 3b9acc
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 20 07:16:17 2026 GMT; NotAfter: Nov 20 07:16:17 2056 GMT
---
Verify return code: 0 (ok)
server: se-201-01 (https)
Host: news.se-201-01.lab
SNI: news.se-201-01.lab
protocol: TLSv1.3
```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера | Pebble Intermediate CA 1e133f |
| Видавець сертифіката видавця | Pebble Root CA 3b9acc |
| Імена в SAN | DNS:www.se-201-01.lab, DNS:news.se-201-01.lab |
| Строк дії (від – до) | v:NotBefore: Nov 20 18:27:45 2026 GMT; NotAfter: Nov 20 18:27:44 2031 GMT та v:NotBefore: Nov 20 07:16:17 2026 GMT; NotAfter: Nov 20 07:16:17 2056 GMT |
| Тип ключа сервера і алгоритм підпису | a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256 та  a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256 |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | 1 (3.5) та 2 (3.6) |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.
Центр Pebble через протокол ACME. lego створив файли challenge, а Pebble перевірив їх GET-запитами.

2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?

3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?

4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?
Старий сертифікат перестане валідуватися, бо Peddle при старті генеруї нові ключі. 

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text
root32mroot@se-201-01:/lab/work/lego/certificates# for i in 1 2 3 4 5; do
  n=f$i.se-201-01.lab; echo "=== $n"
  curl -sS --cacert $STAND_CA https://$n/ 2>&1 | head -1
  openssl s_client -connect $n:443 -servername $n -CAfile $STAND_CA </dev/null 2>/dev/null \
    | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p' | grep -v PKEY
done
=== f1.se-201-01.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f1.se-201-01.lab'
Certificate chain
 0 s:CN = old.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 18 20:27:16 2026 GMT; NotAfter: Feb 16 20:27:16 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 18 20:27:16 2026 GMT; NotAfter: Nov 17 20:27:16 2031 GMT
---
Verify return code: 0 (ok)
=== f2.se-201-01.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f2.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 18 20:27:16 2026 GMT; NotAfter: Feb 16 20:27:16 2027 GMT
---
Verify return code: 21 (unable to verify the first certificate)
=== f3.se-201-01.lab
server: web, f3
Certificate chain
 0 s:CN = f3.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 18 20:27:16 2026 GMT; NotAfter: Feb 16 20:27:16 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 18 20:27:16 2026 GMT; NotAfter: Nov 17 20:27:16 2031 GMT
---
Verify return code: 0 (ok)
=== f4.se-201-01.lab
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f4.se-201-01.lab
   i:CN = f4.se-201-01.lab
   v:NotBefore: Nov 18 20:27:16 2026 GMT; NotAfter: Feb 16 20:27:16 2027 GMT
---
Verify return code: 18 (self-signed certificate)
=== f5.se-201-01.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f5.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: May  2 20:27:16 2026 GMT; NotAfter: Oct 29 20:27:16 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 18 20:27:16 2026 GMT; NotAfter: Nov 17 20:27:16 2031 GMT
---
Verify return code: 10 (certificate has expired)
```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 | (60) SSL: no alternative certificate subject name matches target host name 'f1.se-201-01.lab' | 0 (ok) | Ім'я в SAN | X509v3 Subject Alternative Name: DNS:wrong.lab | Додати доменне ім'я в SAN |
| f2 | (60) SSL certificate problem: unable to get local issuer certificate | 21 (unable to verify the first certificate) | Ланцюжок довіри | error=20:unable to get local issuer certificate | Додати проміжний сертифікат |
| f3 | - | 0 (ok) | Пройдено | Verify return code: 0 (ok) | Нічого |
| f4 | (60) SSL certificate problem: self-signed certificate | 18 (self-signed certificate) | Ланцюжок довіри | error=18:self signed certificate | Замінити сертифікат на підписений довіреним CA |
| f5 | (60) SSL certificate problem: certificate has expired | 10 (certificate has expired) |  Строк дії | error=10:certificate has expired | Оновити сертифікат |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?
Сервер не надсилає проміжний сертифіка. В нас вже зазначений довірений проміжний CA (OPISM Stand Intermediate CA) але він не надісланий. При недовіреному CA зазначається сторонній центр, якого немає в довірених. 

2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.
Бо curl робить повну перевірку звіряючи хост. openssl s_client валідує виключно підписи та дати, але не перевіряє хост з полем SAN сертифіката (якщо не передано -verify_hostname).

3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?
Ключ -k повністю включає перевірку сертифіката клієнтом, тому всі 5 серверів будуть відповідати на цей запит. Порівняно з `--cacert` втрачається автентифікація сервера.

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | 2 RTT | 2 RTT | 1 RTT |
| `time_appconnect` | 3 RTT | 4 RTT | 1 RTT |
| `time_starttransfer` | 4 RTT | 5 RTT | 2 RTT |

Вивід кроків 5.2 і 5.3:

```text
root@se-201-01:/lab/work/lego/certificates# F='%{time_nF='%{time_namelookup} %{time_connect} %{time_appconnect} %{time_starttransfer} %{time_total} %{http_version} %{num_connects}\n'
U=https://origin.se-201-01.lab/

for i in 1 2 3; do curl -s -o /dev/null --cacert $STAND_CA -w "$F" $U; done
for i in 1 2 3; do curl -s -o /dev/null --tls-max 1.2 --cacert $STAND_CA -w "$F" $U; done
curl -s --cacert $STAND_CA -w "$F" -o /dev/null $U -o /dev/null $U -o /dev/null $U
0.035660 0.070854 0.111034 0.146599 0.146721 2 1
0.036113 0.071501 0.110326 0.145720 0.145790 2 1
0.036184 0.071578 0.110754 0.146118 0.146186 2 1
0.035838 0.071102 0.146086 0.181841 0.181979 2 1
0.035923 0.071151 0.146435 0.181848 0.181906 2 1
0.035959 0.071182 0.145683 0.181292 0.181433 2 1
0.035745 0.071040 0.111220 0.146576 0.146654 2 1
0.000018 0.000000 0.000000 0.035417 0.035494 2 0
0.000017 0.000000 0.000000 0.035373 0.035455 2 0
```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | 36.113 | 71.501 | 110.754 |  146.118 | 146.186 | 1 |
| TLS 1.2, нове | 35.923 | 71.151 | 146.086 | 	181.841 | 181.906 | 1 |
| 5.3, перший запит | 35.745 | 	71.040 | 111.220 | 	146.576 | 146.654 | 1 |
| 5.3, другий запит | 0.018 | 0 | 0 | 35.417 | 35.494 | 0 |
| 5.3, третій запит | 0.017 | 0 | 0 | 35.373 | 35.455 | 0 |

RTT, визначений вимірюванням, і з якого значення його визначено: RTT ≈ 35.4 мс (штучна затримка стенда - 35 мс + ~0.3-0.4 мс накладних витрат).

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?

2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?

3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:

## Декларація використання ШІ

Використовувалися інструменти ШІ: Так

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | | | |
