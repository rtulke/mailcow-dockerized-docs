Mails are stored compressed (lz4) and encrypted. The key pair can be found in crypt-vol-1.

If you want to decode/encode existing maildir files, you can use the following script at your own risk:

Enter Dovecot by running `docker-compose exec dovecot-mailcow /bin/bash` in the mailcow-dockerized location.

```
# Decrypt /var/vmail
find /var/vmail/ -type f -regextype egrep -regex '.*S=.*W=.*' | while read -r file; do
if [[ $(head -c7 "$file") == "CRYPTED" ]]; then
doveadm fs get compress lz4:0:crypt:private_key_path=/mail_crypt/ecprivkey.pem:public_key_path=/mail_crypt/ecpubkey.pem:posix:prefix=/ \
  "$file" > "/tmp/$(basename "$file")"
  if [[ -s "/tmp/$(basename "$file")" ]]; then
    chmod 600 "/tmp/$(basename "$file")"
    chown 5000:5000 "/tmp/$(basename "$file")"
    mv "/tmp/$(basename "$file")" "$file"
  else
    rm "/tmp/$(basename "$file")"
  fi
fi
done
```

```
# Encrypt /var/vmail
find /var/vmail/ -type f -regextype egrep -regex '.*S=.*W=.*' | while read -r file; do
if [[ $(head -c7 "$file") != "CRYPTED" ]]; then
doveadm fs put crypt private_key_path=/mail_crypt/ecprivkey.pem:public_key_path=/mail_crypt/ecpubkey.pem:posix:prefix=/ \
  "$file" "$file"
  chmod 600 "$file"
  chown 5000:5000 "$file"
fi
done
```

```
#!/bin/bash

## bash debug mode
#set -x

output_dir="/tmp/var2"
input_dir="/tmp/vmail/"

## dovecot private and public key
#private_key="/mail_crypt/ecprivkey.pem"
#public_key="/mail_crypt/ecpubkey.pem"
private_key="/mail_crypt/ecprivkey_old.pem"
public_key="/mail_crypt/ecpubkey_old.pem"

## dovecot mail compression
lz4compression="1"

#### main

## generate start time
starttime=$(date "+%H:%M")
echo
echo "Start: $starttime $(date "+%p")"

## count all mail files from $input_dir
numfiles=$(find $input_dir -type f -regextype egrep -regex '.*S=.*W=.*'|wc -l)

## start value for all, crypted, decrypted files
i=0
c=0
u=0

find $input_dir -type f -regextype egrep -regex '.*S=.*W=.*'| while read -r file; do

  ## create directory structure only path of file
  extract_path=`dirname "$file"`
  extract_file=`basename "$file"`
  newpath="$output_dir$extract_path"
  newfile="$output_dir$file"

  ## create new dir
  mkdir -p -- "$newpath"
  ### read first bytes of file
  cryptedtext=$(head -c 7 "$file")

  ## encrypt detection
  if [ "$cryptedtext" = "CRYPTED" ]; then
    ## decrypt each mail
    doveadm fs get compress lz4:${lz4compression}:crypt:private_key_path=${private_key}:public_key_path=${public_key}:posix:prefix=/ "$file" > "$newfile"
    c=$(($c+1))
  else
    ## copy plaintext mails
    cp "$file" "$newfile"
    u=$(($u+1))
  fi

  ## file exist and size > 0
  if [ -s "$newfile" ]; then
    ## change file params
    chmod 600 "$newfile"
    chown 5000:5000 "$newfile"
  fi
  i=$((i+1))
  printf "\rProgress: $i/$numfiles, Found encrypted: $c unencrypted: $u"
done


## convert the date "1970-01-01 hour:min:00" in seconds from Unix EPOCH time
stoptime=$(date "+%H:%M")
echo
echo "Stopped: $stoptime $(date "+%p")"

IFS=: read old_hour old_min <<< "$starttime"
IFS=: read hour min <<< "$stoptime"
sec_old=$(date -d "1970-01-01 $old_hour:$old_min:00" +%s)
sec_new=$(date -d "1970-01-01 $hour:$min:00" +%s)

echo "Time elapsed: $(( (sec_new - sec_old) / 60)) minutes"
```
