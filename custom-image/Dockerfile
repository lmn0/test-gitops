# custom-image/Dockerfile
#
# ALTERNATIVE to the initContainer approach used by default in apps/argocd/values.yaml.
#
# The initContainer approach (default) is preferred for this repo because it needs no image
# registry, no rebuild-on-Argo-CD-version-bump maintenance, and is Argo CD's own documented
# pattern for custom tooling. This Dockerfile is provided for teams whose policy requires
# fully pinned, pre-built, scanned images with no runtime downloads (e.g. air-gapped
# clusters, or supply-chain policies that forbid initContainers pulling from the internet
# at pod-start time).
#
# Build & push:
#   docker build -t your-registry/argocd-repo-server:v2.13.2-helm3.7.2 .
#   docker push your-registry/argocd-repo-server:v2.13.2-helm3.7.2
#
# Then in apps/argocd/values.yaml, instead of the initContainers/volumes/volumeMounts block,
# point repoServer.image at this image:
#
#   repoServer:
#     image:
#       repository: your-registry/argocd-repo-server
#       tag: v2.13.2-helm3.7.2

FROM quay.io/argoproj/argocd:v2.13.2 AS base

FROM base
USER root

ARG HELM_VERSION=3.7.2
ARG TARGETARCH=amd64

RUN curl -fsSL https://get.helm.sh/helm-v${HELM_VERSION}-linux-${TARGETARCH}.tar.gz \
      -o /tmp/helm.tar.gz \
    && tar -xzf /tmp/helm.tar.gz -C /tmp \
    && install -m 0755 /tmp/linux-${TARGETARCH}/helm /usr/local/bin/helm \
    && rm -rf /tmp/helm.tar.gz /tmp/linux-${TARGETARCH} \
    && helm version --short   # fails the build if the wrong binary got installed

USER 999
