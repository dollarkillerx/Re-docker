FROM ubuntu:18.04
RUN apt-get update \
    && apt-get install -y curl \
    && rm -fr /var/lib/apt/lists/*
ENTRYPOINT ["curl","-s","https://ip.cn"]
