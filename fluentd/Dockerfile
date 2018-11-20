# or v0.12-onbuild
FROM fluent/fluentd:v1.3.0-onbuild

LABEL Maintainer "Andreas Elser"
LABEL Version "0.1"
LABEL Name "fluentd logger with elasticsearch plugin"

# below RUN includes plugin as examples elasticsearch is not required
# you may customize including plugins as you wish

RUN apk --no-cache --no-progress upgrade \
RUN gem update --system \
    && gem install fluent-plugin-elasticsearch \
    && gem cleanup