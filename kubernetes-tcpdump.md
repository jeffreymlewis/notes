# Running tcpdump in a kubernetes pod
Troubleshooting apps often requires capturing and analysing network traffic. But how can we capture network traffic in containers with no diagnostic tools? One way is to startup a sidecar container in the same pod, containing the diagnostic tools you need. (Note this will restart your pod, which might not be advisable for stateful production services.)

1. Download and install [wireshark](https://www.wireshark.org/download.html) on your laptop
1. install a sidecar container in your pod containing the tools you need. For example, the following container has network diagnostic tools like `tcpdump`. (I make no warranties as to the safety, security, or stability of this container)
```
      - image: nicolaka/netshoot
        imagePullPolicy: IfNotPresent
        name: netshoot
        command:
        - sleep
        - infinity
```
3. Use `tcpdump` to capture network traces of your app, then analyse with wireshark
```
kubectl exec -i pod-name-5b548dc7b5-hdmwz -c netshoot -- tcpdump -i eth0 -w - > pod-name.pcap
^C
wireshark pod-name.pcap
```
4. You can also pipe tcpdump directly to wireshark and see traffic in real time.
```
kubectl exec -i pod-name-5b548dc7b5-hdmwz -c netshoot -- tcpdump -i eth0 -w - | wireshark -k -i -
```
