ARG REGISTRY=localhost
FROM $REGISTRY/oai-build-base:latest.el8 AS builder

ARG GIT_TAG=k8s

WORKDIR /root

RUN git clone --depth=1 --branch=$GIT_TAG --recurse-submodules https://github.com/OPENAIRINTERFACE/openair-cn.git
COPY patches patches/
RUN patch -p1 -d openair-cn < patches/enable_sudo-less_build.patch
RUN rm -rf /usr/local/lib/freeDiameter /usr/local/lib/libfd* \
    && cd openair-cn/build/git_submodules/freeDiameter \
    && git pull origin master \
    && patch -p1 < ../../patches/0001-opencoord.org.freeDiameter.patch \
    && rm -rf build && mkdir -p build && cd build \
    && cmake .. \
    && awk '{if (/^DISABLE_SCTP/) gsub(/OFF/, "ON"); print}' CMakeCache.txt > tmp && mv tmp CMakeCache.txt \
    && make -j`nproc` \
    && make install
RUN cd openair-cn/scripts \
    && ln -sf /usr/local/bin/asn1c_cn /usr/local/bin/asn1c \
    && ln -sf /usr/local/share/asn1c_cn /usr/local/share/asn1c \
    && OPENAIRCN_DIR=$(dirname $(pwd)) ./build_mme -c -v -b Debug\
    && ./check_mme_s6a_certificate ../.. mme.openair4G.eur 


FROM registry.redhat.io/ubi8/ubi
LABEL name="oai-mme" \
      version="$GIT_TAG" \
      maintainer="Frank A. Zdarsky <fzdarsky@redhat.com>" \
      io.k8s.description="openair-cn MME $GIT_TAG." \
      io.openshift.tags="oai,mme" \
      io.openshift.non-scalable="true"

RUN REPOLIST=codeready-builder-for-rhel-8-x86_64-rpms \
    PKGLIST="libconfig libasan libidn lksctp-tools iproute iputils procps-ng bind-utils" && \
    # yum -y upgrade-minimal --setopt=tsflags=nodocs --security --sec-severity=Critical --sec-severity=Important && \
    yum -y install --enablerepo ${REPOLIST} --setopt=tsflag=nodocs ${PKGLIST} && \
    yum -y clean all

ENV APP_ROOT=/opt/oai-mme
ENV PATH=${APP_ROOT}:${PATH} HOME=${APP_ROOT}
RUN mkdir -p -m 777 ${APP_ROOT}/certs
COPY --from=builder /root/openair-cn/build/mme/build/mme ${APP_ROOT}/bin/
COPY --from=builder /root/*.pem ${APP_ROOT}/certs/
COPY --from=builder /usr/local/lib/libfd* /lib64
COPY --from=builder /usr/local/lib/freeDiameter/* /usr/local/lib/freeDiameter/
COPY --from=builder /usr/local/lib/liblfds* /usr/lib64
COPY scripts ${APP_ROOT}/bin/
COPY configs ${APP_ROOT}/etc/
RUN chmod -R u+x ${APP_ROOT} && \
    chgrp -R 0 ${APP_ROOT} && \
    chmod -R g=u ${APP_ROOT} /etc/passwd
#USER 10001
WORKDIR ${APP_ROOT}

# expose ports
EXPOSE 3870/tcp 5870/tcp 2123/udp

VOLUME ["${APP_ROOT}/certs"]

#CMD ["/opt/oai-mme/bin/mme", "-c", "/opt/oai-mme/etc/mme.conf"]
ENTRYPOINT ["/opt/oai-mme/bin/entrypoint.sh"]
CMD ["/bin/bash"]
