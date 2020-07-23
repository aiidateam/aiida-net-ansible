# aiida-net-ansible

Configuration for managing the [aiida.net](http://www.aiida.net/) website

The site is deployed on [AWS](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1), on the `aiidatheos` account and generated *via* [Wordpress](https://wordpress.org/).

To run:

1. Install python and ansible requirements:

```console
pip install -r requirements.txt
ansible-galaxy install -r requirements.yml
```

2. Create an Ubuntu 18.04 VM on AWS (t2.micro and expose port 80)

3. Replace IP in hosts.yml

4. Replace `wordpress_admin_password` and `mysql_dbpassword` in playbook.yml

5. Run the playbook

6. The following tasks are currently manual:

- modify `/etc/apache2/sites-available/vhosts.conf`, to match `files/vhosts.conf`
- `sudo a2dissite 000-default` to disable this config which also runs on port 80
- copy the fluidapp theme to the server and install + activate
  (this is required since fluidapp isn't listed by `wp-cli theme seasrch` and hasn't actually been supported for many years: <https://themeforest.net/item/fluidapp-responsive-mobile-app-wordpress-theme/3726350/comments?page=9>)

```console
scp files/theme-fluidapp.zip /tmp/
ssh <VM_IP>
sudo -u www-data wp-cli --path=/var/www/wp_aiida theme install --activate /tmp/fluidapp.zip
```

- scp files/dbexample.json to /var/www/wp_aiida/dbexample.json (via /tmp/)

- extract user data from latest backup (e.g. from theossrv3), e.g.:

```console
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/mysql_table_dump.mysql
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/var_www/wp_aiida/wp-content/uploads
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/var_www/wp_aiida/wp-content/gallery
```

- then scp them to /var/www/wp_aiida/wp-content/ on VM (via /tmp/)

- Import the backup database:

```console
wp-cli db import --path=/var/www/wp_aiida /tmp/mysql_table_dump.mysql
```

- ensure all folders/files are owned by the wordpress user

```console
sudo chown -R www-data:www-data /var/www/wp_aiida/wp-content/
```

- for testing only (to stop redirect of IP to aiida.net):
  copy files/disable-canonical-redirect.php to /var/www/wp_aiida/wp-content/plugins/disable-canonical-redirect.php (via /tmp/) and activate

```console
wp-cli --path=/var/www/wp_aiida plugin activate disable-canonical-redirect
```

- `sudo systemctl reload apache2`
- `curl -v localhost` to test locally the served front page, and you should be able to access also from a local web-browser, using the VMs IP

## TODO

How to test other pages than front page?
`disable-canonical-redirect.php` allows for testing of front page, but then all links are set to aiida.net.
Gio mentioned using the chrome plugin: Virtual Hosts (to fake hostname), but I haven't had any success with this yet

How to make the new aiida.net point to the new IP?

Gio mentioned: important thing to remember: be careful on issue of swap file. Must create one other mysql will die (500 error)
swapon (already done on quantum mobile cloud edition, search for which role)
(Note I haven't found an issue with this yet)

Move to https: Gio mentioned about role for letsencrypt (DNS authentiction), modify on fly apache config https certificate

There is actually also an error with the site I have noted: that `/var/www/wp_aiida/wp-content/themes/fluidapp/javascripts/aiida_viewer.js` refers to a relative path for `dbexample.json`, dependendant on the current page being served. So it fails for pages other than the front index page.
