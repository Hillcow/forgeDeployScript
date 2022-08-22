# Laravel Forge Zero Downtime Deploy Script
Zero downtime deployment script for Laravel Forge using symlinks that manages releases, static directories and log files. Only deploys if script runs through. Keep x amount of previous successful releases.

### Setup
1. On Laravel Forge at Site/Meta change your `Web Directory` to `/current/public`
2. ssh into site dir
3. Only if you know what you're doing: run the following command that will DELETE EVEVERYTHING of your website EXCEPT your environment file and storage directory:  `find . ! \( -path "./.env" -o -path "./storage*" \) -delete && mkdir -p current && mv .env ./current/.env`
4. upload other possible static folders into site root dir
5. insert deployment script from `./forgeZeroDowntimeDeployScript.sh` into deploy script on Forge at Site/App.
6. Configure the setup variables to your liking:
    - `DOMAIN` = your site name
    - `PROJECT_REPO` = your GIT repo
    - `AMOUNT_KEEP_RELEASES` = any amount of successful releases that you would like to keep at any point of time. Older ones will be deleted after each successful deployment, if you have more than that. `./current/.successes` keeps track of which release was successful (e.g. script ran through at least until symlinking the built release to the `current` dir).
    - `AMOUNT_KEEP_PREVIOUS_LOG_LINES` = On each deployment, all Laravel log files will be put into `*_previous.log` files, so that you have a fresh empty log file on each release. Here you can define the amount of lines that you would like to keep from each previous log file.
    - `STATIC_DIRS` = if you host files (eg. images) yourself, put in paths for symlinking these directories.

### Why?
The purpose of this script is to have a free, easy, secure way of deploying your Laravel application on Forge without any downtime. Without the need for paid services like `Envoyer` or larger applications like `Deployer`.

Advantages compared to the default deployment script provided by Laravel Forge:
- Better user experience: Zero Downtime Deployments
- More secure: Only deploy if the whole script runs through without errors
- Better developer experience: Log management that makes sense

## The script

```shell
declare -A STATIC_DIRS

# SETUP #
DOMAIN=www.your-domain.com
PROJECT_REPO="git@github.com:USERNAME/REPO.git"
AMOUNT_KEEP_RELEASES=5
AMOUNT_KEEP_PREVIOUS_LOG_LINES=500
#STATIC_DIRS["$HOME/$DOMAIN/storage/app"]=storage/app
#STATIC_DIRS["$HOME/$DOMAIN/otherfiles"]=otherfiles

RELEASE_NAME=$(date +%s--%Y_%m_%d--%H_%M_%S)
RELEASES_DIRECTORY=~/$DOMAIN/releases
DEPLOYMENT_DIRECTORY=$RELEASES_DIRECTORY/$RELEASE_NAME

# stop script on error signal (-e) and undefined variables (-u)
set -eu

printf '\nℹ️ Starting deployment %s\n' "$RELEASE_NAME"
mkdir -p "$RELEASES_DIRECTORY" && cd "$RELEASES_DIRECTORY"

printf '\nℹ️ Clone %s, branch "%s"\n' "$PROJECT_REPO" "$FORGE_SITE_BRANCH"
git clone --depth 1 --branch "$FORGE_SITE_BRANCH" "$PROJECT_REPO" "$RELEASE_NAME"
cd "$RELEASE_NAME"

printf '\nℹ️ Copy ./.env file\n'
ENV_FILE=~/"$DOMAIN"/current/.env
if [ -f "$ENV_FILE" ]; then
  cp -pv $ENV_FILE ./.env
else
  printf '\nError: .env file is missing at %s.' "$ENV_FILE" && exit 1
fi

printf '\nℹ️ Copy old ./storage/.log files\n'
cd ./storage/logs
AMOUNT_LOG_FILES=$(find ~/$DOMAIN/current/storage/logs -maxdepth 1 -mindepth 1 -type f -name "*.log" | wc -l)
if [ "$AMOUNT_LOG_FILES" != 0 ]; then
  find ~/$DOMAIN/current/storage/logs -maxdepth 1 -mindepth 1 -type f -name "*.log" -exec cp -pv '{}' '.' ';'
  for filename in *.log; do
    PREVIOUS_LOG=$filename
    if [[ $filename != *"_previous.log" ]]; then
      PREVIOUS_LOG="${filename%.log}"_previous.log
      if [ -f "$PREVIOUS_LOG" ]; then
        printf 'Appending old log file %s content into %s.\n' "$filename" "$PREVIOUS_LOG"
        cat "$filename" >> "$PREVIOUS_LOG"
        rm "$filename"
      else
        printf 'Rename old log file to %s.\n' "${filename%.log}"_previous.log
        mv -- "$filename" "${filename%.log}"_previous.log
      fi
    fi
    BEFORE=$(wc -l < "$PREVIOUS_LOG")
    if [ "$BEFORE" -gt "$AMOUNT_KEEP_PREVIOUS_LOG_LINES" ]; then
      printf 'Cutting %s from %s lines down to your configured max amount %s.\n' "$PREVIOUS_LOG" "$BEFORE" "$AMOUNT_KEEP_PREVIOUS_LOG_LINES"
      echo "$(tail -"$AMOUNT_KEEP_PREVIOUS_LOG_LINES" "$PREVIOUS_LOG")" > "$PREVIOUS_LOG"
    fi
  done
else
  printf '\nNo log files found.\n'
fi
cd "$DEPLOYMENT_DIRECTORY"

printf '\nℹ️ Link configured static directories\n'
for STATIC_DIR in "${!STATIC_DIRS[@]}"
do
  printf 'Linking %s to %s.\n' "$PWD/${STATIC_DIRS[$STATIC_DIR]}" "$STATIC_DIR"
  if [ -d "$STATIC_DIR" ]; then
    rm -rf "${STATIC_DIRS[$STATIC_DIR]}"
    ln -s -n -f -T "$STATIC_DIR" "${STATIC_DIRS[$STATIC_DIR]}"
  else
    printf '\nError: configured static dir is missing at %s.' "$STATIC_DIR" && exit 1
  fi
done

printf '\nℹ️ Install Composer Dependency Updates\n'
$FORGE_COMPOSER install --no-interaction --prefer-dist --optimize-autoloader --no-dev

printf '\nℹ️ Installing NPM dependencies based on \"./package-lock.json\"\n'
npm ci
printf '\nℹ️ Generating JS App files\n'
npm run prod

printf '\nℹ️ Laravel Artisan commands\n'
if [ -f artisan ]; then
  printf '\nℹ️ Link ./public/storage\n'
  $FORGE_PHP artisan storage:link

  printf '\nℹ️ Clear and cache routes, config, views, events\n'
  $FORGE_PHP artisan config:cache
  $FORGE_PHP artisan route:cache
  $FORGE_PHP artisan view:cache
  $FORGE_PHP artisan event:cache

  printf '\nℹ️ Database Migrations\n'
  $FORGE_PHP artisan migrate --force

  printf '\nℹ️ Restart Horizon Queue Workers\n'
  $FORGE_PHP artisan horizon:terminate
else
  printf '\nError: "%s artisan" missing.' "$FORGE_PHP" && exit 1
fi

printf '\nℹ️ !!! Link Deployment Directory !!!\n'
echo "$RELEASE_NAME" >> $RELEASES_DIRECTORY/.successes
if [ -d ~/$DOMAIN/current ] && [ ! -L ~/$DOMAIN/current ]; then
  rm -rf ~/$DOMAIN/current
fi
ln -s -n -f -T "$DEPLOYMENT_DIRECTORY" ~/$DOMAIN/current

printf '\nℹ️ Restart PHP FPM\n'
( flock -w 10 9 || exit 1
    echo 'Restarting FPM...'; sudo -S service $FORGE_PHP_FPM reload ) 9>/tmp/fpmlock

# Clean Up
cd $RELEASES_DIRECTORY

printf '\nℹ️ Delete failed releases:\n'
if grep -qvf .successes <(ls -1); then
  grep -vf .successes <(ls -1)
  grep -vf .successes <(ls -1) | xargs rm -rf
else
  echo "No failed releases found."
fi

printf '\nℹ️ Delete old successful releases:\n'
AMOUNT_KEEP_RELEASES=$((AMOUNT_KEEP_RELEASES-1))
LINES_STORED_RELEASES_TO_DELETE=$(find . -maxdepth 1 -mindepth 1 -type d ! -name "$RELEASE_NAME" -printf '%T@\t%f\n' | head -n -"$AMOUNT_KEEP_RELEASES" | wc -l)
if [ "$LINES_STORED_RELEASES_TO_DELETE" != 0 ]; then
  find . -maxdepth 1 -mindepth 1 -type d ! -name "$RELEASE_NAME" -printf '%T@\t%f\n' | sort -t $'\t' -g | head -n -"$AMOUNT_KEEP_RELEASES" | cut -d $'\t' -f 2-
  find . -maxdepth 1 -mindepth 1 -type d ! -name "$RELEASE_NAME" -printf '%T@\t%f\n' | sort -t $'\t' -g | head -n -"$AMOUNT_KEEP_RELEASES" | cut -d $'\t' -f 2- | xargs -I {} sed -i -e '/{}/d' .successes
  find . -maxdepth 1 -mindepth 1 -type d ! -name "$RELEASE_NAME" -printf '%T@\t%f\n' | sort -t $'\t' -g | head -n -"$AMOUNT_KEEP_RELEASES" | cut -d $'\t' -f 2- | xargs rm -rf
else
  AMOUNT_KEEP_RELEASES=$((AMOUNT_KEEP_RELEASES+1))
  LINES_STORED_RELEASES_TOTAL=$(find . -maxdepth 1 -mindepth 1 -type d -printf '%T@\t%f\n' | wc -l)
  printf 'There are only %s successfully stored releases, which is less than or equal to your\ndefined %s releases to keep, so none of them got deleted.' "$LINES_STORED_RELEASES_TOTAL" "$AMOUNT_KEEP_RELEASES"
fi

printf '\nℹ️ Status - stored releases:\n'
find . -maxdepth 1 -mindepth 1 -type d -printf '%T@\t%f\n' | sort -nr | cut -f 2-

printf '\n✅ Deployment DONE: %s\n' "$DEPLOYMENT_DIRECTORY"
```

### Make it better
Feel free to create a PR or an issue if you feel you can enhance this script.