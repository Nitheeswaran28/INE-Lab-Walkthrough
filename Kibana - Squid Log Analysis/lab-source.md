# Kibana: Squid Log Analysis source notes

INE/AttackDefense walkthrough 1190 supplies the findings and Kibana filters. Results were checked against the PDF, with screenshots used to resolve conflicting prose. The raw proxy log was not supplied.

- Page 4 shows 192.161.225.4 for Q2. The closing prose on the next page transposes part of that IP.
- Page 10 shows a subnet wildcard for Q4 and three matches. The prose query lists just one address. The screenshot also shows a port-21 result on .4, while the prose attributes it to .5. The .1 entry is an aborted result with status 000.
- Pages 19 and 20 show the admin proxy user and the 28,202 ms authenticated tunnel.
- The walkthrough preserves the PDF's search strings, except where it uses the subnet filter shown in the Q4 screenshot.

[AttackDefense challenge 1190](https://attackdefense.com/challengedetails?cid=1190)
