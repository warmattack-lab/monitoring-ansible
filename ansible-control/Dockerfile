FROM python:3.12-alpine

ENV PYTHONUNBUFFERED=1
ENV ANSIBLE_FORCE_COLOR=1
ENV DEBIAN_FRONTEND=noninteractive

RUN apk add --no-cache \
      openssh-client \
      sshpass \
      git \
      bash && \
    pip install --no-cache-dir ansible

WORKDIR /workspace

CMD ["/bin/bash"]
