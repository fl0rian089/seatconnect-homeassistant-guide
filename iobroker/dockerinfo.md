i prefer to create multiple docker networks, if you want to run more than one iobroker

on your docker host:
```
 docker network create --driver=bridge iobroker-<anyname>_network
```
continue with the [guide](https://github.com/fl0rian089/seatconnect-homeassistant-guide/blob/58223eda4045d80024186594b40357eddc0c332c/README.md)
