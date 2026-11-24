# Kibana: DNS Log Analysis source notes

INE/AttackDefense walkthrough 1191 supplies the filters and expected answers. The lab credits Mike Sconzo's SecRepo MACCDC 2012 DNS dataset and identifies the dataset license as CC BY 4.0.

I checked 427,935 tab-separated records in the upstream gzip. The top query had 39,273 records. The leading clients had 860 empty-query records, 7,471 NXDOMAIN records, 75,943 total records, and 7,283 IPv4 reverse lookups, respectively. There were 440 AXFR records. The time.apple.com TXT record with connection ID CmjiklOm3bnHgctw contains the answer You've been pwned. The raw DNS log stays outside the repository.

[Original compressed DNS log](https://www.secrepo.com/maccdc2012/dns.log.gz)
