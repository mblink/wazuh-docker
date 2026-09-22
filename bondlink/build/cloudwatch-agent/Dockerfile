FROM debian:latest AS build
ARG LOG_STREAM_PREFIX
ARG LOG_GROUP_NAME
ARG REGION

RUN apt-get update && \
    apt-get install -y ca-certificates curl jq && \
    rm -rf /var/lib/apt/lists/* && \
    mkdir /etc/cwagentconfig

COPY ./config/cwagent.json /root/cwagent_tmpl.json
# Fixed: the 4.10 fork's jq filter wrapped ${VAR} in single quotes, so bash
# never interpolated it and the agent silently fell back to the hardcoded
# template values. Using --arg passes the build args through properly.
# Applied to every collect_list entry (master + worker alerts), not just
# [0]: each entry's templated log_stream_name is the node suffix, rendered
# as "${LOG_STREAM_PREFIX}_<node>" (e.g. prodops02_master), all in one group.
RUN jq --arg prefix "$LOG_STREAM_PREFIX" --arg group "$LOG_GROUP_NAME" --arg region "$REGION" \
      '.logs.logs_collected.files.collect_list |= map(.log_stream_name = "\($prefix)_\(.log_stream_name)" | .log_group_name = $group) | .agent.region = $region' \
      /root/cwagent_tmpl.json > /etc/cwagentconfig/cwagent.json
# Fixed: the 4.10 fork hardcoded the debian/amd64 package, which fails to
# install on arm64 hosts (e.g. Apple Silicon dev/test machines) -- dpkg's
# own architecture name matches AWS's package path naming directly.
RUN ARCH="$(dpkg --print-architecture)" && \
    curl -O "https://s3.amazonaws.com/amazoncloudwatch-agent/debian/${ARCH}/latest/amazon-cloudwatch-agent.deb" && \
    dpkg -i -E amazon-cloudwatch-agent.deb && \
    rm -rf /tmp/* && \
    rm -rf /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard && \
    rm -rf /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl && \
    rm -rf /opt/aws/amazon-cloudwatch-agent/bin/config-downloader

FROM scratch

COPY --from=build /tmp /tmp
COPY --from=build /etc/cwagentconfig/cwagent.json /etc/cwagentconfig/cwagent.json
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
COPY --from=build /opt/aws/amazon-cloudwatch-agent /opt/aws/amazon-cloudwatch-agent

ENV RUN_IN_CONTAINER="True"
ENTRYPOINT ["/opt/aws/amazon-cloudwatch-agent/bin/start-amazon-cloudwatch-agent"]
