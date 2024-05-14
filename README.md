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

Since I have already set up the domain name, I can get a free certificate using Certbot.

Certbot will install a LetsEncrypt SSL certificate for free.

Ensure your domain name has propagated before running CertBot.

Your domain and IP will be different from mine, and note that it may take some time for the DNS record to propagate across the internet.

On my server, I will run

```bash
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

Now we can run CertBot.

```bash
sudo certbot --nginx
```

Follow the prompts and select the domain name I want to secure.

Next open the Nginx Prometheus config file we created earlier to see the changes.

```bash
sudo nano /etc/nginx/sites-enabled/prometheus
```


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

```bash
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

- Prometheus: [link](http://127.0.0.1:9090/metrics)
- Node Exporter: [link](http://127.0.0.1:9100/metrics)

## Scrape Targets Basics

[Instructor Notes](https://sbcode.net/prometheus/scrape-targets/)

In Prometheus, a "scrape target" refers to any system or endpoint that provides metrics data in a format that Prometheus can fetch and store. These targets are essentially instances that Prometheus polls at regular intervals to collect metrics, which are then used for monitoring and alerting.

### Key Concepts

1. **Endpoints**: A scrape target is typically an HTTP endpoint that exposes metrics in a format Prometheus understands—usually as plaintext in a simple line-based format where each line contains a metric, its value, and optionally, a set of labels.

2. **Scrape Configuration**: Prometheus uses a configuration file (usually called `prometheus.yml`) where you define the details of scrape targets. These details include the frequency of scraping (how often Prometheus collects data), the specific endpoints to scrape, and any additional parameters like timeout limits or authentication details.

3. **Job and Instance**: In the context of scraping:
   - **Job**: A job is a collection of similar scrape targets that share a common purpose. For example, you might have a job to scrape metrics from all instances of a microservice in your infrastructure.
   - **Instance**: Each target within a job is called an instance. If your job is to monitor multiple replicas of a service, each replica would be an instance.

### Example Configuration

Here's a simple example of a scrape configuration in a Prometheus `prometheus.yml` file:

```yaml
scrape_configs:
  - job_name: 'example-job'
    scrape_interval: '15s'  # How often Prometheus scrapes metrics
    static_configs:
      - targets: ['192.168.1.1:9090', '192.168.1.2:9090']  # List of scrape targets
```

In this example:
- `job_name` defines the name of the job.
- `scrape_interval` specifies that Prometheus should scrape metrics every 15 seconds.
- `targets` are the actual endpoints Prometheus will hit to pull metrics.

### How It Works

During operation, Prometheus regularly hits the endpoints specified under each job's `targets`. Each target needs to expose metrics in a way that Prometheus can parse. This is often facilitated by client libraries available for various programming languages that automatically format the metrics correctly.

When Prometheus scrapes a target, it pulls the data and stores it internally, using a time series format. This data is then available for querying using Prometheus's query language (PromQL), and it can be visualized in dashboards using tools like Grafana.

## Delete time series data

There may be a time when you want to delete data from the Prometheus `TSDB` database. Data will be automatically deleted after the storage retention time has passed. By default, it is 15 days. If you want to delete specific data earlier, then you are able. You need to enable the admin API in Prometheus before you can.

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Add the following lines to the configuration file.

```yaml
ARGS= "--web.enable-admin-api"
```

Save and exit the file.

Restart Prometheus.

```bash
sudo service prometheus restart
```

Check status.

```bash
sudo service prometheus status
```

You can now make calls to the admin API.In my example I want to delete all time series for the instance="sbcode.net:9100" So I run the delete_series API endpoint providing the value to match. E.g.,

```bash
curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]={instance="sbcode.net:9100"}'
```

So I run the delete_series API endpoint providing the value to match. E.g.,

```bash
curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]={instance="sbcode.net:9100"}'
```

When I re-execute the Prometheus query, the time series I wanted to be deleted no longer exists.

You can have more complicated match queries, e.g.,

```bash
curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=a_bad_metric&match[]={region="mistake"}'

curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=node_exporter:memory:percent'

curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=node_filesystem_free_percent'

curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=node_memory_MemFree_percent'

curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=ALERTS'

curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=ALERTS_FOR_STATE'
```

After deleting you can disable the admin API by removing the line from the configuration file.

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Remove the line.

```yaml
ARGS= "--web.enable-admin-api"
```

Restart and check the status of Prometheus.

```bash
sudo service prometheus restart
sudo service prometheus status
```
