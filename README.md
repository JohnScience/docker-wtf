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

you will get `bin dev etc home lib media mnt opt proc root run sbin srv sys tmp usr var` instead of `*` as the output. This could be a problem if you want to pass the `--allowed-origins = *` to [`chromedriver`](https://chromedriver.chromium.org/) by default and parameterize the argument via the [`ARG`](https://docs.docker.com/engine/reference/builder/#arg) instruction.

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

## Do I really have to use `ENV ENV_VAR=smth` instead of `RUN export ENV_VAR=smth`?

* Problem

The `RUN export ENV_VAR=smth` doesn't work. It is a no-op. The `ENV` instruction is the only way to set the value of an environment variable.

```Dockerfile
FROM alpine
RUN export PORT=80
ENTRYPOINT [ "sh", "-c", "echo $PORT" ]
```

It will print an empty line.

You can verify it with the following command,

```console
docker build -f Dockerfile.006 -t wtf . && docker run --rm wtf & docker rmi wtf
```

* Mitigation

Just use the `ENV` instruction where possible.

```Dockerfile
FROM alpine
ENV PORT=80
ENTRYPOINT [ "sh", "-c", "echo $PORT" ]
```

It will print `80`.

You can verify it with the following command,

```console
docker build -f Dockerfile.008 -t wtf . && docker run --rm wtf & docker rmi wtf
```

Beware though that the `ENV` instruction is limited.

* The scripts that alter the value of environment variables might not work as expected.
* You won't be able to set the value of an environment variable conditionally unless you provide it via the `docker run -e` flag.

## I just want to set the value conditionally

* Problem

Setting the value of an environment variable depending on the value of an expression inside Docker image doesn't work. You can only unconditionally set the value with the [`ENV`](https://docs.docker.com/engine/reference/builder/#env) instruction, which is limited.

Even this simple script doesn't work,

```Dockerfile
FROM alpine
# the value of PORT can be changed via the -e flag of `docker run`
ENV PORT=80
RUN if [ "$PORT" = "80" ]; then\
    export IS_DEFAULT=1; \
    else \
    export IS_DEFAULT=0; \
    fi
ENTRYPOINT [ "sh", "-c", "echo IS_DEFAULT=$IS_DEFAULT" ]
```

You can verify it with the following command,

```console
docker build -f Dockerfile.005 -t wtf . && docker run --rm wtf & docker rmi wtf
```

It will print `IS_DEFAULT=`.

* Mitigation

The best solution I've come up with is using files. For example,

```Dockerfile

```

## ARG and ENV are a joke

* Problem

Consider this simple script,

```Dockerfile
FROM alpine
ARG PORT=80
ENTRYPOINT ["echo", "$PORT"]
```

I would love `ENTRYPOINT ["echo", $PORT]` to work but it doesn't. However, if we run the original script like this,

```console
docker build -f Dockerfile.002 -t wtf . && docker run --rm wtf & docker rmi wtf
```

it will always print `$PORT` instead of the value of the `PORT` variable.

Alright, let's try to fix it this way,

```Dockerfile
FROM alpine
ARG PORT=80
ENTRYPOINT ["sh", "-c", "echo $PORT"]
```

Now we should see `80` as the output, right? Wrong! We will see an empty line instead.

You can verify it by running the following command,

```console
docker build -f Dockerfile.003 -t wtf . && docker run --rm wtf & docker rmi wtf
```

Let's try the idiomatic way.

```Dockerfile
FROM alpine
ARG PORT=80
ENV PORT=$PORT
ENTRYPOINT ["sh", "-c", "echo $PORT"]
```

It works! You can verify it by running the following command,

```console
docker build -f Dockerfile.004 -t wtf . && docker run --rm wtf & docker rmi wtf
```

Now we can just use the `ENV` instruction to parameterize the `ENTRYPOINT` instruction, right? Wrong! There is an awesome interaction betwee
