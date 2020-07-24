# aiida-net-ansible

Configuration for managing the [aiida.net](http://www.aiida.net/) website

The site is deployed on [AWS](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1), on the `aiidatheos` account and generated *via* [Wordpress](https://wordpress.org/).

To run:

1. Install python and ansible requirements:

```console
pip install -r requirements.txt
ansible-galaxy install -r requirements.yml
```

2. Create an Ubuntu VM on AWS (18.04, t2.micro and expose port 80)

3. Replace IP in hosts.yml

4. Replace `wordpress_admin_password` and `mysql_dbpassword` in playbook.yml

5. Extract user data from latest backup into `backup` folder (e.g. from theossrv3), e.g.:

```console
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/mysql_table_dump.mysql
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/var_www/wp_aiida/dbexample.json
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/var_www/wp_aiida/wp-content/uploads
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/var_www/wp_aiida/wp-content/gallery
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/var_www/wp_aiida/wp-content/themes/fluidapp
```

6. Run the playbook: `ansible-playbook playbook.yml`

7. The following tasks are currently manual:

- `sudo a2dissite 000-default` to disable this config which also runs on port 80

- (optional) ensure all folders/files are owned by the wordpress user

```console
sudo chown -R www-data:www-data /var/www/wp_aiida/wp-content/
```

- (optional) for testing only (to stop redirect of IP to aiida.net):
  copy plugins/disable-canonical-redirect.php to /var/www/wp_aiida/wp-content/plugins/disable-canonical-redirect.php (via /tmp/) and activate

```console
wp-cli --path=/var/www/wp_aiida plugin activate disable-canonical-redirect
```

- `sudo systemctl reload apache2` (if necessary)
- `curl -v localhost` to test locally the served front page, and you should be able to access also from a local web-browser, using the VMs IP

## TODO

It is not super clear, the distinction between installation folders/files (i.e. that should only change due to upgrades) and user folders/files (i.e. that will change based on the content and styling of the website). I think above I have identified the only content that needs to be restored, but this should be checked. (for the theme it should probably only be some aspects like custom css and js)

Gio mentioned: important thing to remember: be careful on issue of swap file. Must create one other mysql will die (500 error)
swapon (already done on quantum mobile cloud edition, search for which role)
(Note I haven't found an issue with this yet)

Move to https: Gio mentioned about role for letsencrypt (DNS authentiction), modify on fly apache config https certificate

There is actually also an error with the site I have noted: that `/var/www/wp_aiida/wp-content/themes/fluidapp/javascripts/aiida_viewer.js` refers to a relative path for `dbexample.json`, dependendant on the current page being served. So it fails for pages other than the front index page.

The fluidapp theme isn't listed by `wp-cli theme search` (i.e. can't be directly installed) and hasn't actually been supported for many years: <https://themeforest.net/item/fluidapp-responsive-mobile-app-wordpress-theme/3726350/comments?page=9>.
So ideally we would replace this

## Notes

For working with the test site `new-www.aiida.net`, just change the `ServerName` (in `/etc/apache2/sites-available/vhosts.conf`) to this address,
and also find and replace `www.aiida.net` in the mysql dump (before importing).

On the current server there is a `/var/www/aiida` folder, but that only contains a minimal site index with the message:
"Dear users, starting from today 29 October 2013 and for the next few days the AiiDA home site will be offline for updates and a new setup, more contents and features will be soon available."
