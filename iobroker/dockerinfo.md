i prefer to create multiple docker networks, if you want to run more than one iobroker

on your docker host:
```
 docker network create --driver=bridge iobroker-<anyname>_network
```
continue with the [guide](https://github.com/fl0rian089/seatconnect-homeassistant-guide/blob/19b6942bb7ca6d367960255cc2ebefc1f0d6963a/README.md)
