# Apache Log Analysis Basics source notes

INE/AttackDefense walkthrough 103 supplies the original pipelines. Pages 2 through 5 show the record count and rankings. Pages 5 and 6 show response-status and client filtering examples.

The original dataset URL returned HTTP 404 during verification. The archived copy contains 2,660,050 records. Independently parsing its first 10,000 lines reproduced the PDF screenshots' counts for client IPs, user agents, complete request lines, and paths. Field-aware analysis found 108 distinct request targets with a non-200 response. Client 91.141.1.150 made 139 requests across 109 distinct targets.

The original Q6 pipeline rejects lines containing 200 anywhere. The original Q7 pipeline searches the whole line and sorts ascending. The walkthrough keeps those pipelines and shows exact-field adaptations.

- [Original dataset URL](http://www.almhuette-raith.at/apache-log/access.log)
- [Archived access log](https://github.com/jlcanela/spark-hands-on/blob/master/almhuette-raith.log/access.log.gz)
