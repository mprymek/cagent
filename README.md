 ```
   _____                  _                                _
  / ____|                | |         /\                   | |
 | |     _ __ _   _ _ __ | |_ ___   /  \   __ _  ___ _ __ | |_
 | |    | '__| | | | '_ \| __/ _ \ / /\ \ / _` |/ _ \ '_ \| __|
 | |____| |  | |_| | |_) | || (_) / ____ \ (_| |  __/ | | | |_
  \_____|_|   \__, | .__/ \__\___/_/    \_\__, |\___|_| |_|\__|
               __/ | |                     __/ |
              |___/|_|                    |___/

                         B - E - T - A !
              suggestions and patches are welcome

 Convenient one-file {en|de}cryption tool using ssh-agent RSA keys.
```

Features
--------
  * {en|de}crypt files using ssh-agent-stored keys (you don't have
       to write any passwords, just hit Enter when ssh-agent asks
       you to use a key (you use "-c" with ssh-add, don't you?)
  * ssh-agent can even store your smartcard PIN (see -s in man ssh-add)
    and use your non-exportable RSA keys
  * store your encrypted config files on your webserver and conveniently
    run commands using them while keeping the config-decrypted-time
    at the bare minimum! (better then ever-mounted encfs or such)

Intro
-----
Cagent uses openssl for AES-256 encryption. Different
key is used for every file/run. The key is derived this way:

```
          key = sha256(ssh_agent_sign(random_salt))
```

ssh_agent_sign = salt is signed with the chosen RSA private key
kept in your ssh-agent. ssh-agent provides only sign operation,
so we must use it instead of a normal RSA encryption.

For every plaintext file _pfile_ cagent creates two files:
  * pfile.enc - AES256 encrypted pfile
  * pfile.meta - json formatted metadata = salt and RSA key fingerprint

Note: if you lose your RSA key, there's no way to retrieve the files content!
Keep a backup of the key in a secure location.

Requirements
------------
   * working ssh-agent holding at least one key
   * openssl
   * python
   * Paramiko python library (for communication with ssh-agent)
   * (optional but strongly advised) secure tmp directory - i.e. some directory where you can put your decrypted files temporarily = encrypted partition, ramdisk (with encrypted or disabled swap!) etc.

Configuration
-------------
```
# ssh-add -l
2048 aa:bb:cc:dd:ee:ff:00:11:22:33:44:55:66:77:88:99 /usr/lib64/opensc-pkcs11.so (RSA)

# vim /usr/local/bin/cagent
## set at least:
##     default_key_fp="aa:bb:cc:dd:ee:ff:00:11:22:33:44:55:66:77:88:99"
```

Usage
-----
(with verbose=True)

     # mkdir priv pub
     # chmod 700 priv
     # echo "This is a testfile" >priv/testfile

     ### encryption

     # cagent e priv/testfile pub/
     * Key found, signing salt             <------ ssh-agent will ask you for permission to sign
     * Running: /usr/bin/openssl aes-256-cbc -in priv/testfile -out pub/testfile.enc -pass stdin
     # ls pub
     testfile.enc  testfile.meta
     # cat pub/testfile.meta
     {"salt": "rTL9wzkb2x6U786zPx1W95sC8VZd+jY2PAzBivRGhCU=", "key": "aa:bb:cc:dd:ee:ff:00:11:22:33:44:55:66:77:88:99"}

     ### decryption

     # rm priv/testfile
     # cagent d pub/testfile.enc priv/
     * Key found, signing salt
     * Running: /usr/bin/openssl aes-256-cbc -d -in pub/testfile.enc -out priv/testfile -pass stdin
     # cat priv/testfile
     This is a testfile

     ### using temporary decrypted file in command

     # cagent r pub/testfile.enc cat FILE
     * Key found, signing salt
     * Running: /usr/bin/openssl aes-256-cbc -d -in pub/testfile.enc -out /privtmp/tmppB9jwM/testfile -pass stdin
     * Running: cat /privtmp/tmppB9jwM/testfile
     This is a testfile
     * Command exited with: 0

     ### real life example: put encrypted Bacula Bat config on the web and run Bat using it!

     # cagent r https://www.mydomain.com/cfg/bacula/bat.conf.enc bat -c FILE
     * Fetching https://www.mydomain.com/cfg/bacula/bat.conf.enc
     * Fetching https://www.mydomain.com/cfg/bacula/bat.conf.meta
     * Key found, signing salt
     * Running: /usr/bin/openssl aes-256-cbc -d -in /privtmp/tmppw6pqx/bat.conf.enc -out /privtmp/tmppw6pqx/bat.conf -pass stdin
     * Running: bat -c /privtmp/tmppw6pqx/bat.conf
     * Removing decrypted cfg file         <-------- config file is removed few seconds after command is run

                       ######## command still running, config is in memory only
                       ######## no decrypted config on disk

     * Command exited with: 0
