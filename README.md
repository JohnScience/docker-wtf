# Docker, WTF?

Weird Docker behaviors and tricks.

## Table of Contents

* [I just want to write a star](#i-just-want-to-write-a-star)
* [Do I really have to use `ENV ENV_VAR=smth` instead of `RUN export ENV_VAR=smth`?](#do-i-really-have-to-use-env-env_varsmth-instead-of-run-export-env_varsmth)
* [I just want to set the value conditionally](#i-just-want-to-set-the-value-conditionally)
* [You can't use the value of ARG directly](#you-cant-use-the-value-of-arg-directly)
* [Naming ENV and ARG varibles the same way may yield unexpected results](#naming-env-and-arg-varibles-the-same-way-may-yield-unexpected-results)
* [Appendix A. Miscellaneous](#appendix-a-miscellaneous)

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

* Mitigation 1

If possible, just quote the build `ARG`,

```Dockerfile
FROM alpine
ARG ALLOWED_ORIGINS="*"
RUN echo "$ALLOWED_ORIGINS" > /allowed-origins.txt
ENTRYPOINT ["sh", "-c", "cat /allowed-origins.txt"]
```

The output will be `*`.

You can verify it with the following command,

```console
docker build -f Dockerfile.010 -t wtf . && docker run --rm wtf & docker rmi wtf
```

* Mitigation 2 (deprecated)

If you find a reason for this mitigation to be useful, please let me know.

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

Even this simple script

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

prints `IS_DEFAULT=` instead of `IS_DEFAULT=1`.

You can verify it with the following command,

```console
docker build -f Dockerfile.005 -t wtf . && docker run --rm wtf & docker rmi wtf
```

It will print `IS_DEFAULT=`.

* Mitigation

The best solution I've come up with is using files. For example,

```Dockerfile
FROM alpine
ENV PORT=80
RUN if [ "$PORT" = "80" ]; then\
    echo "1" > is_default.txt; \
    else \
    echo "0" > is_default.txt; \
    fi
ENTRYPOINT [ "sh", "-c", "echo IS_DEFAULT=$(cat /is_default.txt)" ]
```

correctly prints `IS_DEFAULT=1`.

For why we use files intead of environment variables, see [I just want to set the value conditionally](#i-just-want-to-set-the-value-conditionally).

## You can't use the value of ARG directly

* Problem

Consider this simple script,

```Dockerfile
FROM alpine
ARG PORT=80
ENTRYPOINT ["echo", "$PORT"]
```

I would love `ENTRYPOINT ["echo", $PORT]` to work but it doesn't. However, if we run the script like this,

```console
docker build -f Dockerfile.002 -t wtf . && docker run --rm wtf & docker rmi wtf
```

it will always print `$PORT` instead of `80` (the value of the `PORT` variable).

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

It will print `80`, as expected! You can verify it by running the following command,

```console
docker build -f Dockerfile.004 -t wtf . && docker run --rm wtf & docker rmi wtf
```

## Naming ENV and ARG varibles the same way may yield unexpected results

* Problem

With some remarks,

```Dockerfile
ARG PORT=80
ENV PORT=$PORT
```

is a useful pattern for setting the default value of an environment variable for the container while allowing to override the defaults with [`--build-arg`](https://docs.docker.com/build/guide/build-args/) or [`-e`](https://docs.docker.com/engine/reference/run/#environment-variables) flags.

Let's try to run the following command with several different combination of the `--build-arg` and `-e` flags,

```Dockerfile
FROM alpine
ARG PORT=80
ENV PORT=$PORT
ENTRYPOINT ["sh", "-c", "echo $PORT"]
```

1. No overriding

```console
docker build -f Dockerfile.004 -t wtf . && docker run --rm wtf & docker rmi wtf
```

As expected, it prints `80`.

2. Overriding the `PORT` build-time (`ARG`) variable

```console
docker build --build-arg PORT=8080 -f Dockerfile.004 -t wtf . && docker run --rm wtf & docker rmi wtf
```

As expected, it prints `8080`.

3. Overriding the `PORT` environment (`ENV`) variable

```console
docker build -f Dockerfile.004 -t wtf . && docker run --rm -e PORT=8080 wtf & docker rmi wtf
```

As expected, it prints `8080`.

4. Overriding both the `PORT` build-time (`ARG`) and environment (`ENV`) variables

```console
docker build --build-arg PORT=8080 -f Dockerfile.004 -t wtf . && docker run --rm -e PORT=8081 wtf & docker rmi wtf
```

As expected, the value of the `PORT` environment variable prevails and `8081` is printed.

**However**, if `ARG` and `ENV` variables are named the same way, you may get unexpected results.

If you run the Dockerfile below,

```Dockerfile
FROM alpine
ARG PORT=80
ENV PORT=$PORT
RUN echo $PORT > /port.txt
ENTRYPOINT [ "sh", "-c", "cat /port.txt" ]
```

and try to override the `PORT` environment variable,

```console
docker build -f Dockerfile.009 -t wtf . && docker run --rm -e PORT=8080 wtf & docker rmi wtf
```

you will still get `80` instead of `8080`. The `PORT` environment variable **does** get overriden but in the line

```Dockerfile
RUN echo $PORT > /port.txt
```

the `PORT` variable refers to the `ARG` variable, not the `ENV` variable.

* Mitigation

The best solution I've come up with is to name the `ARG` and `ENV` variables differently. At the moment of writing, there is no convention for discriminating between `ARG` and `ENV` variables but I can recommend naming them either `PORT_DEFAULT` and `PORT` or `PORT_ARG` and `PORT`.

## RUN instruction captures the value of the environment variable at the time of the build

Flagging the `docker run` with the `-e` flag *usually* allows to override the value of the environment variable set in the `Dockerfile` with the `ENV` instruction. It is its distinctive feature compared to the build `ARG`.

* Problem

The `RUN` instruction captures the value of the environment variable at the time of the build and not at the time of the run.

Consequently, the following script

```Dockerfile
FROM alpine
ENV PORT="80"
RUN echo "$PORT" > /port.txt
ENTRYPOINT ["sh", "-c", "cat /port.txt"]
```

will always print `80`, regardless of whether you run it with the `-e` flag or not.

So even with this command,

```console
docker build -f Dockerfile.011 -t wtf . && docker run -e PORT=8080 --rm wtf & docker rmi wtf
```

it will print `80` instead of `8080`.

* Mitigation

WIP

## Appendix A. Miscellaneous

<details>
  <summary>Commands to check the contents of Dockerfile.NNN</summary>
  
If you want to see the content of the `Dockerfile.NNN`, you can run the following command,

* Windows

```console
type Dockerfile.NNN
```

* Linux and macOS

```console
cat Dockerfile.NNN
```

where `NNN` is the number of the Dockerfile, e.g. `000`.
  
</details>

<details>
  <summary>TODOs</summary>
  
* Consider adding a section about the aggressiveness of caching in Dockerfile's layers.

* Consider adding recommendations for how to fix the semantics of Dockerfile's instructions.

* Compare various Dockerfile linters to see if they can catch the problems described in this document and if it's a good idea to recommend one or a few of them.
 
</details>
