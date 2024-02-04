# Docker, WTF?

Weird Docker behaviors and tricks.

* [Dockerfile.000]

Script:

```console
docker build -f Dockerfile.000 -t wtf . && docker run --rm wtf & docker rmi wtf
```

Explanation:

For some reason, if you write something like this,

```Dockerfile
FROM alpine
ARG ALLOWED_ORIGINS="*"
RUN echo $ALLOWED_ORIGINS > /allowed-origins.txt

CMD ["sh", "-c", "cat /allowed-origins.txt"]
# ENTRYPOINT [ "sh", "-c", "chromedriver --whitelisted-ips=172.17.0.1 --allowed-origins=*" ]
```

you will get `bin dev etc home lib media mnt opt proc root run sbin srv sys tmp usr var` instead of `*` as the output. This could be a problem if you want to pass the `--allowed-origins = *` to `chromedriver`.
