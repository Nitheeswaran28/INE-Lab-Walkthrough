# ELK: SSH Logs source notes

Source is INE/AttackDefense walkthrough 1138. The PDF remains the original reference. These notes record the page groups and results.

| Question | PDF pages | Answer reported by the manual |
| --- | --- | --- |
| 1 | 2-3 | 3 clients: 192.148.107.6, 192.148.107.7, and 192.148.107.8. |
| 2 | 3-6 | 192.148.107.6. |
| 3 | 6-7 | 3 servers. |
| 4 | 8-12 | 192.148.107.6 made the most attempts against the busiest server, 192.148.107.4. |
| 5 | 13 | 2 modes: password and public key. |
| 6 | 14-15 | 192.148.107.8. |
| 7 | 15 | 3603. |
| 8 | 15-17 | administrator. |
| 9 | 17-20 | 16:17:58 in the manual's display. |
| 10 | 20-23 | 192.148.107.3. |

Page 5 screenshots show `ssh-logs*` and `sshd_client_ip.keyword`, which differ from the prose. Pages 19–20 show the peak in the Kibana time series. The manual does not specify a separate ingestion-time field.

[AttackDefense challenge 1138](https://attackdefense.com/challengedetails?cid=1138)
