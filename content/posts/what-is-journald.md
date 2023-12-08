---
title: "What Is Journald"
date: 2023-12-08T21:06:35+01:00
draft: false
---

# What is journald?
It's a logging framework from systemd. You can use it with `journalctl`. If you just use `journalctl` without
paramters you will get an excerpt of all logs.

## -f
You can follow the new logs that are incoming using the `-f` parameters: `journalctl -f`. It's like `tail -f` for files.

## --since
To filter the logs you can use parameters like `--since`. Example: `journalctl --since="2020-01-01"` then it will give you the
logs that exists since that date.

## --until
Another filter possibility, like `since`. Example: `journalctl --until="2023-01-01"` will give you all logs until the 01.01.2023.

# Configuration
The Logs are usually deleted after a restart. You can make them persistent. Check the configuration `/etc/systemd/journald.conf`.
The default value for `Storage` is `auto`, which means that if the location is existing, the logs will be saved. The 
default location is `/var/log/journal`. But in the default settings there will be used 10% of the total capacity for log files.
This configuration can be edited as ones like. Don't forget to restart the service using `systemd`.