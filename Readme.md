## Servers Public IPs info: 

**WP Servers:**   
http://wp-0.miliia.test.monitoring.intellij.net - 3.77.200.134  
http://wp-1.miliia.test.monitoring.intellij.net - 18.194.251.242  
http://wp-2.miliia.test.monitoring.intellij.net - 54.93.244.237  
http://wp-3.miliia.test.monitoring.intellij.net - 52.59.188.203  
http://wp-4.miliia.test.monitoring.intellij.net - 3.76.216.37  
http://wp-5.miliia.test.monitoring.intellij.net - 18.185.79.93  
http://wp-6.miliia.test.monitoring.intellij.net - 63.180.182.103  
http://wp-7.miliia.test.monitoring.intellij.net - 18.192.129.74  
http://wp-8.miliia.test.monitoring.intellij.net - 18.199.144.92  
http://wp-9.miliia.test.monitoring.intellij.net - 3.76.223.135  

**Monitoring server:**  
monitoring.miliia.test.monitoring.intellij.net - 54.93.189.67

# Ansible
IMPORTANT: add private key `id_rsa.pem` in `ansible/` dir.  

Run ansible playbook to prepare Word Press Servers:
```
cd ansible/
ansible-playbook prepare_wp_servers.yaml -i inventory.ini 
```
This script will install:
- Docker
- Node Exporter (systemd)
- MySQL Exporter (docker)
- PHP-FPM Exporter (docker)
- Nginx Exporter (docker)  
- Wordpress Exporter (docker)

Run ansible playbook to prepare Monitoring Servers:
```
cd ansible/
ansible-playbook prepare_monitoring_servers.yaml -i inventory.ini 
```
This script will install:
- Prometheus
- Grafana

# Urls:
|       Metrics       |             Url                  | 
|---------------------|----------------------------------|
| Node Exporter       | http://127.0.0.1:9100/metrics    |
| MySQL Exporter      | http://127.0.0.1:9104/metrics    |
| PHP-FPM Exporter    | http://127.0.0.1:9253/metrics    |
| Nginx Exporter      | http://127.0.0.1:9113/metrics    |
| WordPress Exporter  | http://127.0.0.1:11011/metrics   | 

**Prometheus:** http://54.93.189.67:9090   
**Grafana:** http://54.93.189.67:3000


# Troubleshooting:
## WordPress Nginx
1\. I checked the provided URL - http://wp-0.miliia.test.monitoring.intellij.net - and noticed that it responded slowly in the browser, taking approximately 30–50 seconds. I was investigating the services running on the server and found that nginx is active, while apache2 has failed.

I reviewed the configuration file `/etc/nginx/nginx.conf` and examined the errors in `/var/log/nginx/error.log`. I attempted to resolve an issue related to `php-fpm.sock`. It may be necessary to use `php8.1-fpm` and update the socket reference in `/etc/nginx/sites-enabled/default` from `php-fpm.sock` to `php8.1-fpm.sock`.

I also considered that the slow response could be related to the default PHP-FPM pool settings in `/etc/php/8.1/fpm/pool.d/www.conf`:
```
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 4
pm.max_spare_servers = 8
```
Reference: [Determining the correct number of child processes for PHP-FPM](https://www.kinamo.be/en/knowledge-base/determining-the-correct-number-of-child-processes-for-php-fpm)  
However, adjusting these parameters did not improve performance. 

Since I confirmed the WordPress installation in `/var/www/html`, I decided to focus on Nginx and planned to install **Nginx**, **WordPress**, **MySQL** and **PHP-FPM** Exporters.

2\. To monitor server metrics, I also decided to install **Node Exporter**.

3\. After setting up monitoring for the Nginx and Node dashboards and observing the system for some time, I noticed a significant increase in CPU load and a high number of active Nginx connections starting from **9 AM**.

I checked the NGINX access log `/var/log/nginx/access.log` to identify the source of requests to the WordPress application and found a large number of GET requests from the IP address `3.121.206.244`:

```
cat /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head

408761 3.121.206.244
  3446 172.17.0.5
  3196 3.77.200.134
    47 165.154.233.77
    47 141.255.164.26
    47 101.36.104.242
    41 171.231.5.85
    28 87.116.163.142
    6 3.137.73.221
    4 34.126.116.241
```

To find the specific requests sent from `3.121.206.244`, I ran:
```
grep "3.121.206.244" /var/log/nginx/access.log | awk '{print $7}' | sort | uniq -c | sort -nr | head -50

136267 /wp-login.php
136267 /?p=1
136167 /
```
I assume it's Brute-force attacks.  

To block the traffic I manually added the following rule to all WordPress servers:
```
sudo iptables -A INPUT -s 3.121.206.244 -j DROP

sudo iptables -L -n --line-numbers
```

This successfully resolved the issue: the WordPress site began responding normally, and both CPU load and NGINX active connections decreased.

4\. I also SSH into the server `3.121.206.244` to investigate the source of the attack and found a Docker container running `direvius/yandex-tank:jmeter-latest`. So, this is [Yandex.Tank](https://github.com/yandex/yandex-tank) - extensible open source load testing tool.  

After applying the block rule, the container failed with the following error:
```
RuntimeError: All connection attempts failed for wp-0.miliia.test.monitoring.intellij.net, use {phantom.connection_test: false} to disable it
```

## Apache
The apache2 service failed because it tried to use port 80, which is already occupied (by nginx). To resolve this issue need to update following files and change port 80 -> 8080:
```
# /etc/apache2/ports.conf:
Listen 80 -> Listen 8080

# /etc/apache2/sites-available/000-default.conf:
<VirtualHost *:80> -> <VirtualHost *:8080>
```

After making these changes, restart Apache:
```
sudo systemctl restart apache2
```

Please, take a look `screenshots/` dir.


# Alerts:
1. If CPU usage (node_cpu_seconds_total) > 80% for 5m -> alert
2. If Memory Usage (node_memory_MemTotal_bytes) > 80% for 5m -> alert
3. If Nginx active connections (nginx_active_connections) too high for 5m -> alert
4. Monitor logs `/var/log/nginx/access.log` to take a look how many requests fron unique IPs. If more than 500 request in 1 min -> alert. 
5. Monitor how many requests for `/wp-login.php` > 200 for 1 min -> alert
```
grep "wp-login.php" /var/log/nginx/access.log | wc -l
```
6. If MySQL Queries per second (QPS) (mysql_global_status_queries) too high for 5m -> alert
7. If too many MySQL connections (mysql_global_status_threads_connected) for 5m -> alert


# Documentation:
1. [My Githup repo for Monitoring](https://github.com/mfatkhutdinova/devops-learning/tree/master/Monitoring)  
2. [Installation Docker in Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-22-04)  
3. [Wordpress Exporter Metrics Repo](https://github.com/aorfanos/wordpress-exporter)
4. [MySQL Exporter Metrics Repo](https://github.com/prometheus/mysqld_exporter) 
5. [PHP-FPM Exporter Metrics repo](https://github.com/hipages/php-fpm_exporter)
6. [Nginx Exporter Metrics repo](https://github.com/nginx/nginx-prometheus-exporter)  
7. [Grafana Node Exporter Dashboard Template](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)
8. [Grafana MySQL Exporter Dashboard Template](https://grafana.com/grafana/dashboards/7362-mysql-overview/)
9. [Grafana PHP-FPM Exporter Dashboard Template](https://grafana.com/grafana/dashboards/4912-kubernetes-php-fpm/)   
10. [Grafana Nginx Exporter Dashboard Template](https://github.com/nginx/nginx-prometheus-exporter/blob/main/grafana/dashboard.json)  
11. [Grafana Wordpress Exporter Dashboard Template](https://github.com/aorfanos/wordpress-exporter/blob/main/grafana/wordpress-exporter.json)
