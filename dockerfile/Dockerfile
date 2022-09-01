# Манифест Docker образа.
FROM centos:7

EXPOSE 9200 9300

ENV PATH=/usr/lib:/usr/lib/jvm/jre-11/bin:$PATH

RUN yum install java-11-openjdk -y

COPY elasticsearch-8.3.3-linux-x86_64.tar.gz /
COPY elasticsearch-8.3.3-linux-x86_64.tar.gz.sha512 /

RUN yum install -y perl-Digest-SHA && \
    yum -y install wget && \
    sha512sum -c elasticsearch-8.3.3-linux-x86_64.tar.gz.sha512 && \
    tar -xzf elasticsearch-8.3.3-linux-x86_64.tar.gz && \
    rm elasticsearch-8.3.3-linux-x86_64.tar.gz && \
    rm elasticsearch-8.3.3-linux-x86_64.tar.gz.sha512

COPY elasticsearch.yml /elasticsearch-8.3.3/config/

ENV ES_JAVA_HOME=/elasticsearch-8.3.3/jdk/
ENV ES_HOME=/elasticsearch-8.3.3

RUN groupadd elasticsearch \
    && useradd -g elasticsearch elasticsearch

RUN mkdir /var/lib/logs \
    && chown elasticsearch:elasticsearch /var/lib/logs \
    && mkdir /var/lib/data \
    && chown elasticsearch:elasticsearch /var/lib/data \
    && chown -R elasticsearch:elasticsearch /elasticsearch-8.3.3/
RUN mkdir /elasticsearch-8.3.3/snapshots &&\
    chown elasticsearch:elasticsearch /elasticsearch-8.3.3/snapshots

USER elasticsearch

CMD ["/usr/sbin/init"]
CMD ["/elasticsearch-8.3.3/bin/elasticsearch"]