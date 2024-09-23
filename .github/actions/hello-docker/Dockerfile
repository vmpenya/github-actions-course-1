# FROM alpine:3.19
FROM ubuntu:24.04
COPY entrypoint.sh /entrypoint.sh
COPY cleanup.sh /cleanup.sh
RUN ls -l /
RUN /entrypoint.sh
CMD ["World"]