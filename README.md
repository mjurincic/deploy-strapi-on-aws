# Deploying a Strapi API on AWS (EC2 & RDS & S3)

> Goal: build a HTTPS-secured Strapi API as Headles CMS for managing content with development & staging & prodution enviroments

Make sure you have already skimmed the [Strapi docs](https://strapi.io/documentation/3.x.x) before you start.

## § Create EC2 instance / RDS instance / S3 bucket

> You can skim this section if you are familiar with AWS

### ⊙ EC2
1. Click `Launch Instance` button
2. Choose AMI: `Ubuntu Server 18.04 LTS (HVM), SSD Volume Type`
3. Choose Instance Type: `General purpose, t2.micro`
4. Configure Instance: *as you like*
5. Add Storage: `General Purpose SSD (gp2), 20 GB`
6. Add Tags: Name - i-Ubuntu18.04-DBRI-
-Strapi-CMS(cms.dickeys.com)
7. Configure Security Group:
    * SSH (22) - `My IP` or `Anywhere` or *as you like*
    * HTTP (80) - `Anywhere`
    * HTTPS (443) - `Anywhere`
8. Review: click `Launch` button, then a modal pops up. If you are a:
    * Newbie: Choose `Create a new key pair` named `cms.dickeys.com`, download it as `cms.dickeys.com.pem`
    * Veteran: *as you like*
9. Finally, click `Launch Instances` button

### ⊙ RDS
--------------------------------------(new database creation flow)-------------------------------------------------

Use `Switch to the new database creation flow.` if not already set 
![alt text](https://raw.githubusercontent.com/mjurincic/deploy-strapi-on-aws/master/console.aws.amazon.com_rds.png) 

1. Click `Create database` button
2. Quick create `NO`
3. Engine options
   * Engine type: `PostgreSQL`
   * Version: `PostgreSQL 10.6-R1`
2. Templates
   * Free tier
3. Settings
   * DB instance identifier: `dickeys-strapi-cms`
   * Credentials Settings:
        * Master username: `root`
        * - [x] `Auto generate a password`
4. DB instance size
   * DB instance performance type: `Burstable`
   * DB instance class: `db.t2.micro`
5. Storage
   * Storage type: `General Purpose SSD`
   * Allocated storage: `20 GB`
6. Availability & durability
   * Multi-AZ deployment: `Do not create a standby instance`
7. Connectivity
   * Virtual Private Cloud (VPC): `Default VPC`
   * Additional connectivity configuration
      * Subnet group: `default`
      * Publicly accessible: `Yes` *(that's why you need a super strong password)*
      * VPC security groups: `Create new`
         * New VPC security group name: `dickeys-strapi-cms`
      * Availability zone: `No preference`
      * Databse port: `5432`
8. Additional configuration
   * Database options/
      * Initial database name: `dickeys_strapi_cms`
      * DB parameter group: `*as you like*`
      * Option group: `*as you like*`
      * IAM db authenticationInfo: `*as you like*`
   * Backup
      * - [x] `Enable automatic backups`
      * Backup retention period: `7 days`
      * Backup window: `*choose an idle peroid in your timezone* 06:00 UTC 1 hours`
       * - [x] `Copy tags to snapshots`
   * Performance Insights
      * - [x] `Enable Performance Insights`
      * Retention period `7 days`
      * Master key `default`
   * Monitoring
      * - [x] `Enable Enhanced monitoring`
      * Granularity `60 seconds`
      * Monitoring Role `default`
   * Log exports 
      * - [x] `Postgresql log`
      * - [x] `Upgrade log`
   * Maintenance
      * - [x] `Enable auto minor version upgrade`
      * Maintenance window: `*choose an idle peroid in your timezone* 08:00 UTC 1 hours`
   * Deletion protection
      * - [x] `Enable deletion protection` 
7. Create databases for development & staging modes (GUI recommend: https://dbeaver.io / https://www.navicat.com)
    * Development mode: `dickeys_strapi_cms_development`
    * Staging mode: `dickeys_strapi_cms_staging`
    * Production mode: `dickeys_strapi_cms` *(already exists)*
    
-------------------------------------------(old new database creation flow)----------------------------------------
1. Engine options; `PostgreSQL`
2. Choose use case: `*as you like*`
4. Instance specifications:
    * DB engine version: `PostgreSQL 10.x-R1`
    * DB instance class: `db.t2.micro`
    * Multi-AZ deployment: `No`
    * Storage type: `General Purpose (SSD)`
    * Allocated storage: `20 GB`
    * DB instance identifier: `dickeys-strapi-cms`
    * Master username: root
    * Password: *as you like, recommend https://passwordsgenerator.net*
5. Configure advanced settings
    * Public accessibility: `Yes` *(that's why you need a super strong password)*
    * Database name: `dickeys_strapi_cms`
    * Monitoring & Maintenance window: *choose an idle peroid in your timezone*
    * Click `Create database` button
6. Instance Details panel - Security groups
    * Edit inbound rules: PostgreSQL (5432) - `Anywhere`
7. Create databases for dev & staging modes (GUI recommend: https://dbeaver.io)
    * Development mode: `strapi_dev`
    * Staging mode: `strapi_staging`
    * Production mode: `strapi` *(already exists)*

### ⊙ S3
0. Click `Create bucket` button
1. Name and region
    * Bucket name: *as you like*
2. Configure options: *as you like*
3. Set permissions
    * - [ ] `Block new public ACLs and uploading public objects (Recommended)`
    * - [ ] `Remove public access granted through public ACLs (Recommended)`
    * - [x] `Block new public bucket policies (Recommended)`
    * - [x] `Block public and cross-account access if bucket has public policies (Recommended)`
    * `Do not grant Amazon S3 Log Delivery group write access to this bucket`
4. Review: click `Create bucket` button

## § Point your domain to EC2
Point the A / CNAME records to the EC2's IPv4 Public IP / Public DNS (IPv4), such as:
* Development mode: `development-cms.dickeys.com`
* Staging mode: `staging-cms.dickeys.com`
* Production mode: `cms.dickeys.com`

## § Warm up EC2

### ⊙ Login to EC2 & update
Switch to the directory where the key pair `strapi-cms.pem` locates:

```shell
$ ssh -i strapi-cms.pem ubuntu@<ec2-public-ip>
$ sudo apt update
```

### ⊙ Install Nginx

```shell
$ sudo apt install nginx
```

### ⊙ Install nvm & Node.js & PM2
* Install [nvm](https://github.com/creationix/nvm#install-script):
    ```shell
    $ curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
    ```
* Install Node.js:
    ```shell
    $ nvm install 10
    ```
* Install [PM2](https://pm2.io):
    ```shell
    $ npm i pm2 -g
    ```

### ⊙ Install Certbot
According to https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx :

```shell
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository universe
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install python-certbot-nginx
```

## § Create a local Strapi project

### ⊙ Installation and initialization
Firstly, install Strapi (refer to https://strapi.io/documentation/3.x.x/getting-started/installation.html):

```shell
$ npm i -g strapi@alpha
```

Secondary, create a new project (refer to https://strapi.io/documentation/3.x.x/cli/CLI.html#strapi-new):

```shell
$ strapi new cms.dickeys.com \
   --dbclient=postgres \
   --dbport=5432 \
   --dbhost=<RDS endpoint> \
   --dbname=dickeys_strapi_cms_development \
   --dbusername=<RDS master username> \
   --dbpassword=<RDS password>
```

Finally, **DO NOT** rush to `strapi start` for now.  
There are some preparatory procedures to accomplish.  
(By the way, you may be interested in https://strapi.io/documentation/3.x.x/advanced/usage-tracking.html)

### ⊙ Complete database configuration
> If you prefer best practices like https://12factor.net/config and https://github.com/i0natan/nodebestpractices/blob/master/sections/projectstructre/configguide.md , you'd better use tools like [dotenv](https://www.npmjs.com/package/dotenv) instead. Look below for example.

```js
// config/functions/bootstrap.js
'use strict';

/**
 * An asynchronous bootstrap function that runs before
 * your application gets started.
 *
 * This gives you an opportunity to set up your data model,
 * run jobs, or perform some special logic.
 */
// Search and load .env
require('dotenv').config({ path: require('find-config')('.env') });

module.exports = cb => {
  cb();
};
```



`strapi-cms/config/environments/development/database.json` has been completed during the initialization.  
Complete `strapi-cms/config/environments/{staging|production}/database.json` based on it. For example:

```js
// config/environments/development/database.json
{
  "defaultConnection": "default",
  "connections": {
    "default": {
      "connector": "strapi-hook-bookshelf",
      "settings": {
        "client": "postgres",
        "host": "${process.env.DATABASE_HOST || '127.0.0.1'}",
        "srv": "${process.env.DATABASE_SRV || false}",
        "port": "${process.env.DATABASE_PORT || 5432}",
        "database": "${process.env.DATABASE_NAME || 'dickeys_strapi_cms_development'}",
        "username": "${process.env.DATABASE_USERNAME || 'postgres'}",
        "password": "${process.env.DATABASE_PASSWORD || 'root'}"
      },
      "options": {
      }
    }
  }
}
```

### ⊙ Fix listening ports
Modify `port` in `strapi-cms/config/environments/{staging|production}/server.json`:
* development: ${process.env.PORT || 1337} (default)
* staging: ${process.env.PORT || 1338}
* production: ${process.env.PORT || 1339}

### ⊙ Install S3 upload plugin
> Refer to https://strapi.io/documentation/3.x.x/guides/upload.html#install-providers

```shell
$ npm i -S strapi-provider-upload-aws-s3@alpha
```

### ⊙ Add npm scripts for staging & production mode
> You can use [PM2 - Ecosystem File](https://pm2.io/doc/en/runtime/guide/ecosystem-file/) instead if you'd like to

`npm start` is for development mode by default.  
You can set `NODE_ENV` before it according to https://strapi.io/documentation/3.x.x/guides/deployment.html .  
However, for the sake of compatibility, [cross-env](https://www.npmjs.com/package/cross-env) is introduced.

```shell
$ npm i -D cross-env
```

So `strapi-cms/package.json` may look like:

```js
{
  ...
  "scripts": {
    "setup": "cd admin && npm run setup",
    "start": "node server.js",                         // for development mode
    "staging": "cross-env NODE_ENV=staging npm start", // for staging mode 
    "prod": "cross-env NODE_ENV=production npm start", // for production mode
    "strapi": "node_modules/strapi/bin/strapi.js",
    "lint": "node_modules/.bin/eslint api/**/*.js config/**/*.js plugins/**/*.js",
    "postinstall": "node node_modules/strapi/lib/utils/post-install.js"
  },
  ...
}
```

### ⊙ Add [`nginx.conf`](https://github.com/61FINTECH/deploy-strapi-on-aws/blob/master/nginx.conf) to the project root
> If you prefer best practices using `/etc/nginx/{sites-available|sites-enabled}`, you may need help from https://nginxconfig.io or https://github.com/h5bp/server-configs-nginx . I prefer single file `nginx.conf` because of simplicity.

Don't forget to replace all `yourdomain.com` with yours.

### ⊙ Run
```shell
$ cd strapi-cms && strapi start
```

Your browser will open http://localhost:1337 automatically later.  
Create the admin with a strong password.

### ⊙ Complete S3 settings
Visit [PLUGINS > Files Upload](http://localhost:1337/admin/plugins/upload/configurations/development) to complete the S3 settings for all modes.

### ⊙ GraphQL
If you'd like to equip GraphQL, visit [GENERAL > Marketplace](http://localhost:1337/admin/install-plugin) to download.  
Refer to https://strapi.io/documentation/3.x.x/guides/graphql.html for further info.

### ⊙ Configure response & security
Since Strapi will be running behind a well-tuned Nginx, you should:
* Visit [GENERAL > Configurations > ENVIRONMENTS > Response](http://localhost:1337/admin/plugins/settings-manager/response/production), set *Production* & *Staging*'s *Gzip* to `OFF`
* Visit [GENERAL > Configurations > ENVIRONMENTS > Security](http://localhost:1337/admin/plugins/settings-manager/security/development), copy all the settings from *Development* to *Production* & *Staging*

> For more details, please turn to https://strapi.io/documentation/3.x.x/configurations/configurations.html

### ⊙ Push to a [Github free private repo](https://blog.github.com/2019-01-07-new-year-new-github/) (or Gitlab, etc)
```json
$ git init
$ git add -A
$ git commit -m 'initialize new strapi project'
$ git remote add origin <private-git-repo-url>
$ git push -u origin master
```

## § Deploy Strapi on EC2

### ⊙ Pull the repo and install npm dependencies
```shell
$ cd ~
$ git clone <private-git-repo-url>
$ cd strapi-cms
$ npm i
```

### ⊙ Run all modes with PM2
Thanks to [this Stack Overflow comment](https://stackoverflow.com/questions/31579509/can-pm2-run-an-npm-start-script/37775318#comment64893202_37775318), you can run npm scripts with PM2:

```shell
$ pm2 start npm --name "strapi-dev" -- start
$ pm2 start npm --name "strapi-staging" -- run staging
$ pm2 start npm --name "strapi-prod" -- run prod
```

Also, you need to set up the [PM2 Startup Hook](https://pm2.io/doc/en/runtime/guide/startup-hook/) in case of reboot:

```shell
$ pm2 startup
$ pm2 save
```

You can check if the startup hook works by rebooting the machine if you'd like to:

```shell
$ sudo reboot

$ ssh -i strapi-cms.pem ubuntu@<ec2-public-ip>
$ pm2 ls
```

### ⊙ Replace `/etc/nginx/nginx.conf` with yours

```shell
$ cd ~/strapi-cms
$ sudo mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
$ sudo cp nginx.conf /etc/nginx/nginx.conf
```

### ⊙ Obtain SSL certificates
According to https://nginxconfig.io :

```shell
# HTTPS - certbot (before first run): create ACME-challenge common directory
$ sudo mkdir -p /var/www/_letsencrypt && chown www-data /var/www/_letsencrypt

# HTTPS - certbot (before first run): disable SSL directives
$ sudo sed -i -r 's/(listen .*443)/\1;#/g; s/(ssl_(certificate|certificate_key|trusted_certificate) )/#;#\1/g' /etc/nginx/nginx.conf

# Reload Nginx config
$ sudo systemctl reload nginx

# HTTPS - certbot: obtain certificates
$ sudo certbot certonly
    --webroot -n --agree-tos --force-renewal \
    -w /var/www/_letsencrypt \
    --email <your-email> \
    -d cms.<yourdomain.com> \
    -d staging-cms.<yourdomain.com> \
    -d dev-cms.<yourdomain.com>

# HTTPS - certbot (after first run): enable SSL directives
$ sudo sed -i -r 's/#?;#//g' /etc/nginx/nginx.conf

# Reload Nginx config again
$ sudo systemctl reload nginx
```

All done! Visit https://{cms|staging-cms|dev-cms}.yourdomain.com to see if it works.

### ⊙ Further optimizations
* See https://strapi.io/documentation/3.x.x/guides/deployment.html
* Remove `strapi-cms/public/index.html`

## § Summary
Now you have:
* A Strapi API running on dev & staging & prod mode simultaneously
* PM2-guarded processes with reboot startup hooks
* Forced HTTPS-secured traffic for all
* Auto-renew SSL certificates for free
