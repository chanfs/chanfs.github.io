---
layout: post
title:  "Querying DKIM with dig"
date:   2021-08-02 14:00:55 +0800
categories: messaging
---
## DKIM record of a domain
There are sites that offer online tools that allows you to query the DKIM record of a domain. In case you wonder how it is done, you can do the same with *dig*, the DNS lookup utility. 

## Using dig to find the DKIM record
```
dig selector2._domainkey.example.com.sg txt

; <<>> DiG 9.16.1-Ubuntu <<>> selector2._domainkey.example.com.sg txt
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42366
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;selector2._domainkey.example.com.sg. IN TXT

;; ANSWER SECTION:
selector2._domainkey.example.com.sg. 1800    IN TXT "v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG7w0BAQEFAAOCAQ8AMIIBCgKCAQEAtPY8SFb1uSaUxDsZ2zCMzCs/USUYQuWlvfyjatFr9/v/HE/GN/uTrpxcN0xmdmjywGMBf39nWLt2qxA2hOz6ERoZ7eLw6WTSGUr0JYr8YFr5uAAnFHt0NsZjjLJf8kt2MfU4yuGZhn0XnCKLmAJ0cgco75VXeI6/qQu5byNqKAD8YHxRqR9qd1T4jPSw1eJ8S" "hd/xlJI7qdcN4h8QhmTM99Ntihx5J02LRcMoGjLL/CqfAqW0AAny+rtzg11MtNKFU2LmG0UMPwPmIEcSSGAp70lVXJed/XoYNirtZYCrEoMCtKQ/V2y2BX+rhjDzErQesjWvLIOGoTd8WDqo45evwIDAQac"

;; Query time: 56 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Sat Jul 31 21:51:17 +08 2021
;; MSG SIZE  rcvd: 494
```

The format is ```dig selector._domainkey.domain``` You must append _domainkey after the selector.

Here you can see that in the ANSWER section, the DKIM record. 

The v=DKIM1 means it is using rfc6376 compliant version of DKIM. This is the only version so far.

k=rsa. This is the key type, which is RSA by default.

p=MIIBIjA....   This is the public-key data (base64).  An empty value means that this public key has been revoked. 

You can find the selector in the email header you received from the domain you are checking. The relevant part of the header will look something like this: 
```DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
        d=example.com.sg; s=selector2;```



