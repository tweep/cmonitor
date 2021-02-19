
FROM ubuntu:18.04


# Build with 
# cd cmonitor  
#
# docker build -f docker/ubuntu/Dockerfile -t cmonitor:latest .  
#
# docker run -it -v "/tmp:/tmp"  cmonitor:latest 
#
# make sure you did run the "cmonitor_musl" target before building this image: 
#RUN apk add libstdc++ libc6-compat build-base bash

RUN apt-get update -y 
RUN apt-get install -y libstdc++6 build-essential libc6-i386 bash 
 
COPY $PWD . 
RUN  make && cp  src/cmonitor_collector /usr/bin/
CMD "/bin/bash"
 
# finally run the cmonitor collector 
#  - in foreground since Docker does not like daemons
#  - do not collect "cgroups": in this way we just collect baremetal performance stats
#  - for this example collect 3minutes of data (60 samples) and then stop (override the ARG to change this)
#  - put resulting files in /perf folder which is actually a volume shared with the host (see docker run command)

#CMD  /usr/bin/cmonitor_collector --sampling-interval=3  --num-samples=0  --collect=cpu,memory,disk,network  --output-directory /tmp && /bin/sh

