# Dockermoodle fork

Here are additional installation instructions for setting up a dockermoodle.

## Steps

Before you start, make sure to have Docker desktop installed and enabled the WSL as Virtual Machine Manager and also "integration with my default WSL distro" enabled.

Start Docker desktop and go into the Linux terminal of choice (for Windows users it is the WSL (`wsl.exe`)). All of the following is done inside Linux environment.

If you like, you can include your Windows git configuration into the WSL. Just make sure to not have the auto CRLF feature in there... (in case of this use `git config --global core.autocrlf input` to overwrite)
```bash
#git config --global --unset include.path
git config --global include.path "/mnt/c/Users/<username>/.gitconfig"
```

You can read this [guide](https://lmstutorials.com/tutorials/moodle_docker/moodle_docker.php) to get familiar with the setup steps and mybe then even skip my steps listed below.

1. Fork and clone this repo **or** fork and clone the original one [here](https://github.com/moodlehq/moodle-docker), remove local.yml from gitignore and then proceed
2. Create a branch for your project and switch to it. It is wise to name it after the moodle version you want to clone and maybe the plugins you want to use or develop...
3. Set environment variables:
- `export MOODLE_DOCKER_DB=pgsql`
- `export MOODLE_DOCKER_WWWROOT=./moodle/...` so name after version so for instance `./moodle/405` so you can have multiple dockermoodles within the same repo; check with `echo $MOODLE_DOCKER_WWWROOT`
4. Clone moodle into this repo; for example `git clone -b MOODLE_403_STABLE git://git.moodle.org/moodle.git $MOODLE_DOCKER_WWWROOT`
5. Add the git excludes (see [below](#git-exclude))
6. Create `config.php` for the moodle setup by using the template file: `cp config.docker-template.php $MOODLE_DOCKER_WWWROOT/
config.php`
7. Create volumes (not bind mounts) using `local.yml` (see [below](#`local.yml`)) &rarr; Note that the `local.yml` only ever lives in branches that are not the main one; they could be ignored or excluded but for this repo fork structure it is better to include them as long as it is done on different branches so we can have mutliple dockermoodle instances without cloning the repo over and over again...
In the yaml also specify a project name and then set the env variable `MV`, for instance `export MV=405full`
8. Spin up the compose project by running `bin/moodle-docker-compose up -d` (`-d` means "detach" so we dont block the terminal)
9. Do the moodle setup
```bash
bin/moodle-docker-compose exec webserver php /var/www/html/admin/cli/install_database.php \
    --agree-license \
    --fullname="Docker moodle" \
    --shortname="docker_moodle" \
    --summary="Docker moodle site" \
    --adminpass="test" \
    --adminemail="admin@example.com"
```
10. Start developing plugins or install some (for installation via browser drag & drop see [below](#permissions))


## Git exlude
- ~~Use Git exlude (`.git/info/exclude`) for this repo and add `local.yml`~~
- Use Git exclude for the cloned moodle repo to add `*` wildcard to it

## `local.yml`
This file can be set up to include stuff for our dockermoodle that is specific to our needs.

Important is to add a name to the compose project so we can distinguish between different moodle versions.

```yml
name: moodledocker${MV:-default}
```

Also add services if you want plugins like STACK or CodeRunner to work.

Also volumes must be added to persist the application data and the database data:

```yml
services:
  webserver:
      volumes:    
        - moodledata:/var/www/moodledata
    db:
      volumes:
        - moodledb:/var/lib/postgresql/data

volumes:
  moodledata:
  moodledb:
```

## Permissions

For uploading plugin zips the webserver user `www-data` is there. But this user doesnt have write permissions for the moodle data.

To fix this, set the group owner and then modify permissions so that the group owner and user owner can modify (`770` for dirs, `660` for files).

```bash
sudo chgrp -R 33 "$MOODLE_DOCKER_WWWROOT"
sudo find "$MOODLE_DOCKER_WWWROOT" -type d -exec chmod 770 {} +
sudo find "$MOODLE_DOCKER_WWWROOT" -type f -exec chmod 660 {} +
```

You can always revert back:

```bash
sudo find "$MOODLE_DOCKER_WWWROOT" -type d -exec chmod 750 {} +
sudo find "$MOODLE_DOCKER_WWWROOT" -type f -exec chmod 640 {} +
```

See also [here](https://moodle.org/mod/forum/discuss.php?d=405387).

## Useful commands in the webserver

**Upgrade**
```bash
bin/moodle-docker-compose exec -T webserver php admin/cli/upgrade.php --non-interactive
```

**Purge caches**
```bash
bin/moodle-docker-compose exec -T webserver php admin/cli/purge_caches.php
```

**Tests**
```bash
bin/moodle-docker-compose exec -T webserver php admin/tool/behat/cli/init.php
```
```bash
bin/moodle-docker-compose exec -T webserver php admin/tool/behat/cli/run.php --tags='@<pluginname>'
```
```bash
bin/moodle-docker-compose exec webserver vendor/bin/phpunit --filter <name>
```

**Ping**
```bash
bin/moodle-docker-compose exec webserver curl <some-url>
```

**Codechecker**
```bash
git clone https://github.com/moodlehq/moodle-local_codechecker.git local/codechecker
```

## Some more thoughts

Well you could also do the simple variant by just cloning the original repo and foillowing the steps of the tutorial. This is fine for most cases but if you want another moodle bversion you have to again clone and do the stuff. Another alternative would be to always upgrade your moodle instance. Then the simple approach is probably better than my suggestion of having some more moodles under the `moodle` dir...