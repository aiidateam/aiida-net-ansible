# aiida-net-ansible

Configuration for managing the [aiida.net](http://www.aiida.net/) website.

The site is deployed on [AWS](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1), on the `aiidatheos` account and served *via* the [Wordpress](https://wordpress.org/) content management system, itself built on the LAMP (Linux, Apache, MySQL, PHP) web service stack.

To run:

1. Install python and ansible requirements:

```console
pip install -r requirements.txt
ansible-galaxy install -r requirements.yml
```

2. Create an Ubuntu VM on AWS (18.04, t2.micro and expose port 80)

3. Replace IP in `hosts.yml` file

4. Replace `wordpress_admin_password` and `mysql_dbpassword` in `playbook.yml` file

5. Extract user data from latest backup into `backup` folder (e.g. from theossrv3), e.g.:

```console
tar -xzf wordpress_backup-2021-01-08.tar.gz wordpress_backup/mysql_table_dump.mysql
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/var_www/wp_aiida/dbexample.json
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/var_www/wp_aiida/wp-content/uploads
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/var_www/wp_aiida/wp-content/gallery
tar -xzf wordpress_backup-2020-07-14.tar.gz wordpress_backup/var_www/wp_aiida/wp-content/themes/fluidapp
```

6. zip the backup folders, e.g.

```console
zip -r theme-fluidapp.zip fluidapp/*
zip -r uploads.zip uploads/*
zip -r gallery.zip gallery/*
```

7. Run the playbook: `ansible-playbook playbook.yml`

8. The following tasks are currently manual:

- `sudo a2dissite 000-default` to disable this config which also runs on port 80 and overrides the actual site config at `/etc/apache2/sites-available/vhosts.conf` (not sure why/when this is created?)

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

With wordpress, it is not super clear, the distinction between installation folders/files (i.e. that should only change due to upgrades) and user folders/files (i.e. that will change based on the content and styling of the website). I think above I have identified the only content that needs to be restored, but this should be checked. (for the theme it should also probably only be some aspects like custom css and js)

For mysql, Gio mentioned:
> important thing to remember: be careful on issue of swap file. Must create one other mysql will die (500 error)
> swapon (already done on quantum mobile cloud edition, search for which role)

(Note I have not found an issue with this yet though)

Move to https: Gio mentioned about role for letsencrypt (DNS authentiction), modify on fly apache config https certificate. Possible resources:

- https://letsencrypt.org/getting-started/#
- https://www.wpbeginner.com/wp-tutorials/how-to-add-ssl-and-https-in-wordpress/
- https://websitesetup.org/http-to-https-wordpress/
- https://www.reddit.com/r/ansible/comments/9eyyug/is_there_a_way_to_fully_automate_setting_up_ssl/
- https://github.com/systemli/ansible-role-letsencrypt
- https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-ubuntu-18-04

There is an error with the existing site I have noted: that `/var/www/wp_aiida/wp-content/themes/fluidapp/javascripts/aiida_viewer.js` refers to a relative path for `dbexample.json`, which is dependant on the current page being served. So it fails for pages other than the front index page.
(possible fix https://api.jquery.com/jQuery.getJSON/)

The fluidapp theme isn't listed by `wp-cli theme search` (i.e. can't be directly installed) and hasn't actually been supported for many years: <https://themeforest.net/item/fluidapp-responsive-mobile-app-wordpress-theme/3726350/comments?page=9>.
So ideally we would replace this

- <https://www.wpbeginner.com/showcase/best-wordpress-themes/>

apache docs mention about not using `.htaccess` files (https://httpd.apache.org/docs/2.4/howto/htaccess.html)?

> You should avoid using .htaccess files completely if you have access to httpd main server config file. Using .htaccess files slows down your Apache http server. Any directive that you can include in a .htaccess file is better set in a Directory block, as it will have the same effect with better performance.

## Notes

For working with the test site `new-www.aiida.net`, just change the `ServerName` (in `/etc/apache2/sites-available/vhosts.conf`) to this address,
and also find and replace `www.aiida.net` in the mysql dump (before importing).

On the old server there is a `/var/www/aiida` folder, but that only contains a minimal site index with the message:
"Dear users, starting from today 29 October 2013 and for the next few days the AiiDA home site will be offline for updates and a new setup, more contents and features will be soon available."
