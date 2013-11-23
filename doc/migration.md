# Guide - Merge Public GitLab into GitLab 6.2

## 1. Introduction

This guide is based on the fact that you have a default GitLab installation. Which means that:

 * Your gitlab user is `git`.
 * Your gitlab directory is `/home/git/gitlab`.

You might have to adapt commands if you're not using default options.

## 2. Backup

It's useful to make a backup just in case things go south: (With MySQL, this may require granting "LOCK TABLES" privileges to the GitLab user on the database version)


```bash
cd /home/git/gitlab
sudo -u git -H bundle exec rake gitlab:backup:create RAILS_ENV=production
```

> I would even recommend a proper database dump.

Save your config files:

```bash
cd /home/git
mkdir gitlab61-save
tar cvfz save/gitlab61.tar.gz gitlab
mv gitlab/config/database.yml gitlab61-save/
mv gitlab/config/gitlab.yml gitlab61-save/
mv gitlab/config/unicorn.rb gitlab61-save/
```

If you have other config files, remember to save them (they're in the tar file, just in case).

## 3. Stop server

    sudo service gitlab stop


## 4. Remove Public GitLab

    rm -rf /home/git/gitlab

## 5. Fetch GitLab sources

> Note: GitLab still use login page as homepage for anonymous... I would highly suggest that you take a look at [Mike's work](https://git.sphere.ly/staff/publicgitlab/commit/fc9586ac52b893d1a5b5babee2f35d29713db13c) on that, and use his changes. It's your choice.

```bash
# Clone GitLab official repository
sudo -u git -H git clone https://github.com/gitlabhq/gitlabhq.git gitlab

# OR
# Clone GitLab with anonymous access to the dashboard
sudo -u git -H git clone https://git.sphere.ly/staff/publicgitlab.git gitlab

# Go to gitlab dir
cd /home/git/gitlab

# Checkout to stable release
sudo -u git -H git checkout 6-2-stable
```

## 6. Setup your config files

```bash
cd /home/git/
cp gitlab61-save/database.yml gitlab/config
cp gitlab61-save/gitlab.yml gitlab/config
cp gitlab61-save/unicorn.rb gitlab/config
```

Then you need to manually edit your `gitlab/config/gitlab.yml` file and remove the following line (if exists):

    default_projects_limit_private: 5

## 7. Database cleanup

MySQL

```
DROP TRIGGER IF EXISTS pgl_new_user;
DROP TRIGGER IF EXISTS pgl_new_project;
DROP TRIGGER IF EXISTS pgl_update_project;
DELETE FROM users WHERE username = 'guest';
DELETE FROM user_teams WHERE name = 'pgl_reporters';
```

PostgreSQL
```
DROP TRIGGER IF EXISTS pgl_new_user ON users;
DROP TRIGGER IF EXISTS pgl_new_project ON projects;
DROP TRIGGER IF EXISTS pgl_update_project ON projects;
DELETE FROM users WHERE username = 'guest';
DELETE FROM user_teams WHERE name = 'pgl_reporters';
```

## 8. Install Gems

```
cd /home/git/gitlab

# For MySQL (note, the option says "without ... postgres")
sudo -u git -H bundle install --deployment --without development test postgres aws

# Or for PostgreSQL (note, the option says "without ... mysql")
sudo -u git -H bundle install --deployment --without development test mysql aws
```

## 9. Database update and stuff

```
sudo -u git -H bundle exec rake db:migrate RAILS_ENV=production
sudo -u git -H bundle exec rake assets:clean RAILS_ENV=production
sudo -u git -H bundle exec rake assets:precompile RAILS_ENV=production
```

## 10. Start services

```
sudo service gitlab start
sudo service nginx restart
```

Everything should be working fine. Now you can upgrade to 6.3 with the official guide. Sorry about the delay.
Thank you all for using this fork when it was useful, and for your support.