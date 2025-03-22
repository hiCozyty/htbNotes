hint: use Impacket tool 

1. basic scan
```bash
nmap -p- 10.129.107.37 -sV -sC
```
taking forever so took out -p-
```bash
nmap  --min-rate 1000 10.129.107.37 -sV -sC
```
output:
```bash

```


