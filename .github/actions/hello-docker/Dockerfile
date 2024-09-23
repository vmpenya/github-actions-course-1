# FROM alpine:3.19
FROM ubuntu:24.04
COPY cleanup.sh /cleanup.sh
COPY entrypoint.sh /entrypoint.sh
ENTRYPOINT [ "/entrypoint.sh" ] 
CMD [ "World" ]