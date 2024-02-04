# Docker, WTF?

Weird Docker behaviors and tricks.

## I just want to write a star

* Problem

For some reason, if you write something like this,

```Dockerfile
FROM alpine
ARG ALLOWED_ORIGINS="*"
RUN echo $ALLOWED_ORIGINS > /allowed-origins.txt

ENTRYPOINT ["sh", "-c", "cat /allowed-origins.txt"]
# ENTRYPOINT [ "sh", "-c", "chromedriver --whitelisted-ips=172.17.0.1 --allowed-origins=*" ]
```

you will get `bin dev etc home lib media mnt opt proc root run sbin srv sys tmp usr var` instead of `*` as the output. This could be a problem if you want to pass the `--allowed-origins = *` to `chromedriver` by default and parameterize the the argument via the `ARG` statement.

* Check the result of the problematic script

```console
docker build -f Dockerfile.000 -t wtf . && docker run --rm wtf & docker rmi wtf
```

* Mitigation

```Dockerfile
FROM alpine
ARG ALLOWED_ORIGINS="*"
RUN if [ "$ALLOWED_ORIGINS" = "bin dev etc home lib media mnt opt proc root run sbin srv sys tmp usr var" ]; then \
    echo "*" > /allowed-origins.txt; \
    else \
    echo "$ALLOWED_ORIGINS" > /allowed-origins.txt; \
    fi

ENTRYPOINT ["sh", "-c", "cat /allowed-origins.txt"]
# ENTRYPOINT [ "sh", "-c", "chromedriver --whitelisted-ips=172.17.0.1 --allowed-origins=*" ]
```

* Check the result of the mitigation

```console
docker build -f Dockerfile.001 -t wtf . && docker run --rm wtf & docker rmi wtf
```
