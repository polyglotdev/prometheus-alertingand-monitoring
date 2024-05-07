# Prometheus Alerting and Monitoring

[Prometheus Alerting and Monitoring](https://www.udemy.com/course/prometheus/)

## Install Prometheus

```bash
sudo apt update
sudo apt install prometheus
```

This will have installed 2 services being Prometheus and the Prometheus Node Exporter. You can verify there status using the commands. (Press Ctrl-C to exit the status log)

```bash
sudo service prometheus status
sudo service prometheus-node-exporter status
```

Prometheus should now be running.

You can visit it at http://[your IP address]:9090

The installation also created a user called Prometheus. You can see which processes it is running by using the command

```bash
ps -aux | grep prometheus
```

- IP Address: 137.184.226.152
- URL: https://prometheus.domhallan-devops.com/

Tool for looking up `cname` information: https://mxtoolbox.com/CnameLookup.aspx

Save and test the new configuration has no errors

Restart Nginx with:

```bash
nginx -t
sudo service nginx restart
sudo service nginx status
```

Test it by visiting again: [My Prometheus Instance](https://prometheus.domhallan-devops.com/)

Visiting your [IP address](137.184.226.152) directly will still show the default Nginx welcome page. If you don't want this to happen, then you can delete its configuration using the command below.

```bash
rm /etc/nginx/sites-enabled/default
```

You can also generate the CSR with [CSR Generator](https://decoder.link/csr_generator). This a myriad of tools like SSL Checker, CSR Decoder, etc.

Next open the Nginx Prometheus config file we created earlier to see the changes.

```bash
sudo nano /etc/nginx/sites-enabled/prometheus
```

Everything is great so far, but anybody in the world with the internet access and the URL can visit my Prometheus server and see my data.

To solve this problem, we will add user authentication.

We will use Basic Authentication.

SSH onto your server and CD into your /etc/nginx folder.

```bash
cd /etc/nginx
sudo apt install apache2-utils
```

Now we can create a password file. In the command below, I am creating a user called 'admin'.

```bash
htpasswd -c /etc/nginx/.htpasswd admin
```

I then enter a password for the user.

Next open the Nginx Prometheus config file we created.

```bash
sudo nano /etc/nginx/sites-enabled/prometheus
```

And add the two authentication properties in the examples below to the existing Nginx configuration file we have already created.

```nginx
server {
    ...

    #additional authentication properties
    auth_basic  "Protected Area";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass           http://localhost:9090/;
    }

    ...
}
```

Save and test the new configuration has no errors

```bash
nginx -t
```

Restart Nginx

```
sudo service nginx restart
sudo service nginx status
```


Since port 9090 and 9100 are still open, we should block them for external connections.

```bash
iptables -A INPUT -p tcp -s localhost --dport 9090 -j ACCEPT
iptables -A INPUT -p tcp --dport 9090 -j DROP
iptables -A INPUT -p tcp -s localhost --dport 9100 -j ACCEPT
iptables -A INPUT -p tcp --dport 9100 -j DROP
iptables -L
```

>⚠️ Warning
> iptables settings will be lost in case of system reboot. You will need to reapply them manually,
> or you can use a tool like iptables-persistent to save them.

install iptables-persistent

```bash
sudo apt install iptables-persistent
```

This will save your settings into two files called,

`/etc/iptables/rules.v4`

`/etc/iptables/rules.v6`

Any changes you make to the iptables configuration won't be auto saved to these persistent files, so if you want to update these files with any changes, then use the commands,

`iptables-save > /etc/iptables/rules.v4`

`iptables-save > /etc/iptables/rules.v6`

When installing Prometheus using the command,

```bash
apt install prometheus
```

It will automatically set up two metrics endpoints.

- Prometheus : http://127.0.0.1:9090/metrics
- Node Exporter : http://127.0.0.1:9100/metrics