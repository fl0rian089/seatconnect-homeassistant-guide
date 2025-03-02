i prefer to create multiple docker networks, if you want to run more than one iobroker

on your docker host:
```
 docker network create --driver=bridge iobroker-<anyname>_network
```
continue with the [guide](https://github.com/fl0rian089/seatconnect-homeassistant-guide/blob/27c2ecd4c4e95f3363126fd309b064a66e5a5350/README.md)
