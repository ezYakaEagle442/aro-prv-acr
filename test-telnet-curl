- [https://oneuptime.com/blog/post/2026-03-04-red-hat-ubi-podman-rhel-9/view](https://oneuptime.com/blog/post/2026-03-04-red-hat-ubi-podman-rhel-9/view)
- [https://access.redhat.com/articles/4238681](https://access.redhat.com/articles/4238681)
- [https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/building_running_and_managing_containers/assembly_adding-software-to-a-ubi-container_building-running-and-managing-containers](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/building_running_and_managing_containers/assembly_adding-software-to-a-ubi-container_building-running-and-managing-containers)

```sh
FROM registry.access.redhat.com/ubi9/ubi
# Utiliser les repos publics UBI (pas besoin de subscription)
RUN dnf install -y --disablerepo=* \
   --enablerepo=ubi-9-baseos-rpms \
   --enablerepo=ubi-9-appstream-rpms \
   curl wget && \
   dnf clean all
```

```
yum install telnet
```
