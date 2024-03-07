---
title: docker vs local environment
slug: docker-vs-local-environment-z1nnrua
url: /post/docker-vs-local-environment-z1nnrua.html
date: '2023-10-20 11:28:25'
lastmod: '2024-03-07 19:12:28'
toc: true
isCJKLanguage: true
---

## local environment

When installing Kubernetes directly in the local environment, user experience might be compromised. Rather than simply updating the Docker image, installing it could potentially clutter the local environment. If there exists a potential conflict, the installation may not succeed. The isolation of the environment ensures that Tailscale, a networking tool, does not influence the local network.

However, there are some advantages to installing in the local environment. First, executing computations locally, rather than in the Docker container, can reduce computational overhead. Second, this approach allows direct use of GPU CUDA in the local environment, reducing the complexity of installing extra GPU drivers.

## docker in docker

Utilizing Docker to encapsulate the environment can leverage the previously mentioned method for installing the demonstration. This approach provides convenience to users, requiring only an update of the Docker image. However, it is important to note that the use of Docker images may result in a reduction of computational power.

Furthermore, the Sysbox plugin must be installed in the local environment to facilitate Kubernetes' operation within Docker. When GPU requirements arise, the corresponding driver needs to be installed within the Docker container, which necessitates specifying the particular GPU.

This encapsulation strategy, while offering ease and convenience, also presents challenges and potential performance degradation. Further investigation into optimizing this configuration might be beneficial to strike a balance between user convenience and computational efficiency.

‍
