FROM       alpine:latest

MAINTAINER Hans Hüppelshäuser<hans.hueppelshaeuser@ibm.com>

RUN apk add --update openssh

RUN mkdir -p /root/.ssh
ADD id_rsa /root/.ssh
ADD id_rsa.pub /root/.ssh
ADD known_hosts /root/.ssh
RUN chmod 700 /root/.ssh/id_rsa

