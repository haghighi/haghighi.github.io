---
title:  "Running Self-Hosted GitLab Pages behind reverse proxy and in a separate server"
layout: post
summary: "Take a shot at setting up two separate instances of GitLab and running each on a different server. "
last_modified_at: 2019-06-15T09:20:12-05:00
author: Ahmad Haghighi
thumbnail: gitlab
#local_thumbnail: gitlab-pages
categories: tutorial
tags:
  - devops
  - gitlab
  - nginx
  - gitlab
  - gitlab-pages
  - proxy
---

[GitLab Pages](https://about.gitlab.com/product/pages/) is a way to create websites for projects and groups in order to publish documentations, wikis, or any static content. Sometimes, for resource limitation, decreasing the load on the main GitLab instance (if self-hosted), to increase security, or for separating docs and wikis from code, we need to host our GitLab Pages in a separate server. To achieve this, we should have two GitLab instance on two distinct machines: one of them is our main GitLab (a normal GitLab installation) and the other one is an instance only for publishing GitLab Pages. 

*This is a tutorial and provides some technical information and configurations, We assume you are familiar with GitLab installation and GitLab Pages, and already have one GitLab self-managed instance (on-premises or in the cloud) in use.*

## Requirements
* Two GNU/Linux machine, each with networking enabled. In this tutorial we are using Ubuntu 16.04 server.
* Add a [wildcard DNS A record](https://en.wikipedia.org/wiki/Wildcard_DNS_record) in your DNS provider in order to pointing to the host that GitLab Pages runs (for example **docs.example.com**):  

```
*.docs.example.com. 1800 IN A    10.1.1.20
```

Throughout the tutorial, we refer to the server that runs main GitLab as `gitlab-master` and the server that runs GitLab Pages as `gitlab-pages`. We will use the following IP addresses for them:  
* **gitlab-master**: 10.1.1.10 gitlab.example.com  
* **gitlab-pages**: 10.1.1.20 docs.example.com  

## Installing GitLab
We are using GitLab Community Edition (gitlab-ce) Omnibus package on both `gitlab-master` and `gitlab-pages` machines. There are several existing methods for installing GitLab; for detailed information read the [official documentations](https://about.gitlab.com/install).  

## Install and Configure NFS
In this scenario, one of GitLabs generates and manages the Pages and the other (`gitlab-pages`) host and publish them, so we need to share a directory on `gitlab-pages` and make it accessible by Main GitLab instance (installed on `gitlab-master`). We are using [Network File System (NFS)](https://en.wikipedia.org/wiki/Network_File_System) to share and mount directory.

### Install Required Packages
The `gitlab-pages` machine is used as NFS Server and we need to install `nfs-kernel-server` package on that, and on the client side (NFS Client: `gitlab-master`) the `nfs-common` package should be installed.  

**On gitlab-master**:   

```bash
gitlab-master$ sudo apt update
gitlab-master$ sudo apt install nfs-kernel-server
```

**On gitlab-pages**:   

```bash
gitlab-pages$ sudo apt update
gitlab-pages$ sudo apt install nfs-common
```

### Make Required Directories
We're going to share two separate directories using NFS, a shared directory inside of the NFS server and the corresponding directory on the client side to make shared directory available. `/var/opt/gitlab/gitlab-rails/shared/pages` is the default path for GitLab Pages in **GitLab Omnibus** package. Se use that as our shared directory on the `gitlab-pages` machine, and make a new directory on `gitlab-master` to be used as a destination for mounting.

```bash
gitlab-master$ sudo mkdir /mnt/pages
```

**Note**:  
The default ownership of `/var/opt/gitlab/gitlab-rails/shared/pages` is belong to `git:gitlab-www`. Sometimes **gid** and **uid** of the group **gitlab-www** and the user **git** are not the same on `gitlab-master` and `gitlab-pages`, and cause some unusual behaviors! We need to have same  **gid** and **uid** for **gitlab-www** and **git** on both machines (in this situations changing the ownership to `nobody:nogroup` does not help). As our main GitLab instance installed on `gitlab-master`, it's recommended to not touch `gitlab-master`, and instead update  **gid** and **uid** in `gitlab-pages` to corresponding ones on the `gitlab-master`. For example, if **uid** of the user **git** in `gitlab-master` is **998**, we should update **uid** of the user **git** on the `gitlab-pages` to **998** (for more information, read man-page's of the `usermod` and `groupmod` commands). Most of the time machines have the same **id**s and we don't have to change anything. If you changed any **id** on the `gitlab-pages` machine, it's recommended (sometimes necessary) to update/apply permission with new **id**s, by downloading and running [update-permissions](https://gitlab.com/gitlab-org/omnibus-gitlab/raw/master/docker/assets/update-permissions) script as **root**. (*Be careful about running any scripts on your machine, specially when executed with __root__ permission. So read the `update-permissions` script carefully.*)  

**On gitlab-master**:   

```bash
gitlab-master$ ls -ld /mnt/pages
drwxr-x--- 5 git gitlab-www 4096 May 20 16:14 /mnt/pages
```

**On gitlab-pages**:   

```bash
gitlab-pages$ sudo ls -ld /var/opt/gitlab/gitlab-rails/shared/pages
drwxr-x--- 5 git gitlab-www 4096 May 20 11:44 /var/opt/gitlab/gitlab-rails/shared/pages
```

### Configure NFS Server
Now it's time to **export** shared directory on the `gitlab-pages`, open `/etc/exports` file in your text editor with root privileges and update contents based on the following template:

```
directory_to_share    client(share_option1,...,share_optionN)
```

Substitute **client** with your `gitlab-master`'s IP address and **directory_to_share** with the shared directory defined in the previous section.

**Contents of `/etc/export` file for current scenario**:

```
# /etc/export On gitlab-pages
/var/opt/gitlab/gitlab-rails/shared/pages 10.1.1.10(rw,sync,no_root_squash,no_subtree_check)
```

When you are finished making your changes, save and close the file. Then, to make the shared directory available to the client, restart the NFS server, and add firewall rules to allow connections on NFS port 2049 (we use ufw for managing firewall):  

```bash
gitlab-pages$ sudo systemctl restart nfs-kernel-server
gitlab-pages$ sudo ufw allow from 10.1.1.10 to any port nfs
```

### Configure NFS Client
First of all, we are trying to manually mount the shared directory (which exported in the previous stage). If operation was successful, then we update /etc/fstab file to mount remote NFS directory automatically at Boot time. 

```bash
gitlab-master$ mount -v 10.1.1.10:/var/opt/gitlab/gitlab-rails/shared/pages  /mnt/pages
```

Continue if you do not get an error:

```bash
gitlab-master$ df -h | grep pages
10.1.1.10:/var/opt/gitlab/gitlab-rails/shared/pages   98G   50G   44G  54% /mnt/pages
```

**Testing access to shared directory**:

```bash
gitlab-master$ sudo -u git touch /mnt/pages/foo
```

```bash
gitlab-pages$ sudo ls -la /var/opt/gitlab/gitlab-rails/shared/pages/foo
-rw-r--r-- 1 git git 0 Jun  8 13:52 /var/opt/gitlab/gitlab-rails/shared/pages/foo
gitlab-pages$ sudo rm /var/opt/gitlab/gitlab-rails/shared/pages/foo
```

If all the previous commands executed successfully, open `/etc/fstab` file in your text editor and add the following line:

```
10.1.1.10:/var/opt/gitlab/gitlab-rails/shared/pages /mnt/pages nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0
```

If you no longer want the remote directory to be mounted on your system, you can unmount it just like the other common file systems:  

```bash
# Unmount your remote directory 
gitlab-master$ sudo umount /mnt/pages
```

## Configure GitLab Instances
After creating the NFS share on `gitlab-pages` and making it accessible from `gitlab-master`, we should disable GitLab Pages on the Main GitLab and disable the other Omnibus services on the GitLab Pages instance.

### Configure GitLab Pages Server (gitlab-pages)
Open `/etc/gitlab/gitlab.rb` in your text editor on `gitlab-pages` server and update its content like this:

```yaml
# /etc/gitlab/gitlab.rb on gitlab-pages 
external_url 'http://10.1.1.20'
pages_external_url "http://docs.example.com"
gitlab_pages['access_control'] = false
postgresql['enable'] = false
redis['enable'] = false
prometheus['enable'] = false
unicorn['enable'] = false
sidekiq['enable'] = false
gitlab_workhorse['enable'] = false
gitaly['enable'] = false
alertmanager['enable'] = false
node_exporter['enable'] = false
gitlab_rails['auto_migrate'] = false
```

Reconfigure GitLab for applying changes:

```bash
gitlab-pages$ sudo gitlab-ctl reconfigure
```

### Configure Main GitLab (gitlab-master)
In order to configure our Main GitLab instance, we only need to update three options, open  `etc/gitlab/gitlab.rb/` file with your text editor and apply the following changes:  

```yaml
# PARTIAL /etc/gitlab/gitlab.rb on gitlab-master 
. . .
gitlab_pages['enable'] = false
pages_external_url "http://docs.example.com"
gitlab_rails['pages_path'] = "/mnt/pages"
. . .
```

Reconfigure GitLab for applying changes:

```bash
gitlab-master$ sudo gitlab-ctl reconfigure
```

## Using Nginx as Reverse Proxy for Pages 
By default GitLab Omnibus uses an embedded Nginx server (listening on port **80**) for its web GUI and to proxy requests to Pages; thus in addition to port **9080** (default port for GitLab Pages), the port number **80** is in use on the `gitlab-pages` server. Some times we need to assign port number **80** or **443** to services other than GitLab, for example running a HA, load balancing or anything else. In this case we should disable GitLab's Nginx (but only on `gitlab-pages`, if we disable Nginx on the Main GitLab, then we will not be able to use GitLab's Web) and use our own Nginx to redirect GitLab Pages requests to Pages daemon (listening on `localhost:9080`).

**Updated `/etc/gitlab/gitlab.rb` file on `gitlab-pages` with disabled Nginx**:

```yaml
# /etc/gitlab/gitlab.rb on gitlab-pages 
external_url 'http://10.1.1.20'
pages_external_url "http://docs.example.com"
gitlab_pages['access_control'] = false
postgresql['enable'] = false
nginx['enable'] = false
redis['enable'] = false
prometheus['enable'] = false
unicorn['enable'] = false
sidekiq['enable'] = false
gitlab_workhorse['enable'] = false
gitaly['enable'] = false
alertmanager['enable'] = false
node_exporter['enable'] = false
gitlab_rails['auto_migrate'] = false
```

**Nginx configuration** (`/etc/nginx/sites-enabled/pages` file):



```config
upstream gitlab-pages{
    server 127.0.0.1:8090;
  }

server {
  listen 80;
  server_name *.docs.example.com;
  location / {
    proxy_pass http://gitlab-pages;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $http_host;

    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forward-Proto http;
    proxy_set_header X-Nginx-Proxy true;

    proxy_redirect off;

  }
}
```

## Further Reading
After completing all previous steps and reloading the Nginx service, the GitLab Pages will be accessible through `*.docs.example.com` domain. A star (`*`) is a namespace for your username, groupname or projectname (see following table). You can use GitLab Pages as usual and no further changes or configurations is needed. 

| Type of GitLab Pages | The name of the project created in GitLab | Website URL |
| -------------------- | ------------ | ----------- |
| User pages  | username.example.com  | http(s)://username.example.com  |
| Group pages | groupname.example.com | http(s)://groupname.example.com |
| Project pages owned by a user  | projectname | http(s)://username.example.com/projectname |
| Project pages owned by a group | projectname | http(s)://groupname.example.com/projectname |
| Project pages owned by a subgroup | subgroup/projectname | http(s)://groupname.example.com/subgroup/projectname |

> You should strongly consider running GitLab pages under a different hostname than GitLab to prevent XSS attacks.  


* [[GitLab Pages]](https://docs.gitlab.com/ee/user/project/pages/index.html): https://docs.gitlab.com/ee/user/project/pages/index.html
* [[GitLab Pages administration]](https://docs.gitlab.com/ee/administration/pages): https://docs.gitlab.com/ee/administration/pages
* [[Static sites and GitLab Pages domains]](https://docs.gitlab.com/ee/user/project/pages/getting_started_part_one.html): https://docs.gitlab.com/ee/user/project/pages/getting_started_part_one.html

