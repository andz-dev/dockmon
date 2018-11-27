# or v0.12-onbuild
FROM fluent/fluentd:v1.3-onbuild

# Some information
LABEL Maintainer "Andreas Elser"
LABEL Version "0.1"
LABEL Description "fluentd with elasticsearch plugin"

# This section is necessary for import company CA certificates
ARG certfile=./ssl_cert/

# Copy the certification files
COPY ${certfile} /usr/local/share/ca-certificates/

# Update the certificates
RUN update-ca-certificates

# below RUN includes plugin as examples elasticsearch is not required
# you may customize including plugins as you wish
RUN apk add --virtual .build-deps \
       sudo build-base ruby-dev \
       && sudo gem install \
       fluent-plugin-elasticsearch \
       && sudo gem sources --clear-all \
       && apk del .build-deps \
       && rm -rf /var/cache/apk/* \
       && gem cleanup

# Delete the company certificates from the container image
RUN rm -rf /usr/local/share/ca-certificates/ \
       && update-ca-certificates