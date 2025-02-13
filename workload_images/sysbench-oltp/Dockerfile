FROM mysql:5.7
LABEL maintainer="nima.mahmoudi@ualberta.ca"

ENV HOSTNAME=unknown
# runmode can be anything, it will added as postfix. na will be discarded.
ENV postfix=n/a

# FROM https://github.com/BretFisher/sysbench-docker-hpe
# and
# https://github.com/akopytov/sysbench
# FOR TESTING ONLY: DB HAS NO PASSWORD

#######################
# TO USE:
# 1. customize ENV's below
# 2. start container and it will create databases (could take minutes or longer)
#    i.e. docker run --storage-opt size=28G -d --name sysbench bretfisher/sysbench-docker-hpe
# 3. run /opt/sysbench_run.sh to kick off a test run
#    i.e. docker exec -it sysbench bash /opt/sysbench_run.sh

# Lots of docs and scripts at https://github.com/BretFisher/sysbench-docker-hpe

# NOTE: mysql has a VOLUME at /var/lib/mysql
# so you may want to customize the volume storage for that or bind-mount to host
#######################

ARG VCS_REF

LABEL org.label-schema.vcs-ref=$VCS_REF \
      org.label-schema.name="sysbench-docker-hpe" \
      org.label-schema.description="Dockerfile for MySQL and Sysbench to benchmark systems" \
      org.label-schema.vcs-ref=$VCS_REF \
      org.label-schema.vcs-url="https://github.com/BretFisher/sysbench-docker-hpe" \
      org.label-schema.vendor="Bret Fisher" \
      org.label-schema.schema-version="1.0"

##
## install sysbench
##
RUN set -x \
        && apt-get update \
        && apt-get install -y --no-install-recommends git ca-certificates automake libtool \
        make vim pkg-config default-libmysqlclient-dev \
        && git clone https://github.com/akopytov/sysbench.git \
        && cd sysbench && ./autogen.sh && ./configure && make && make install \
        && apt-get purge -y --auto-remove ca-certificates automake libtool make vim pkg-config \
        && rm -rf /var/lib/apt/lists/*


##
## install Python
##
RUN apt-get update
RUN apt-get -y install python3 python3-pip


##
## change these values to your environment and needs
#######################
ENV SYSBENCH_TABLE_SIZE=10000 \
    SYSBENCH_TABLE_COUNT=3 \
    SYSBENCH_NUM_THREADS=1 \
    SYSBENCH_MAX_TIME=20 \
    SYSBENCH_MAX_REQUESTS=10

# mysql_buffersize should be 60-80% of memory
ENV MYSQL_ALLOW_EMPTY_PASSWORD=true \
    MYSQL_DATABASE=sysbench \
    MYSQL_CONFIG=/etc/mysql/mysql.conf.d/mysqld.cnf \
    MYSQL_BUFFERSIZE=1G \
    MYSQL_LOGSIZE=256M \
    MYSQL_FLUSHLOG=1 \
    MYSQL_FLUSHMETHOD=O_DIRECT
#######################

# update mysql config file with our setting above, which means mysql ENV's are set at build time
RUN echo "innodb_buffer_pool_size = ${MYSQL_BUFFERSIZE}" >> ${MYSQL_CONFIG} && \
    echo "innodb_log_file_size = ${MYSQL_LOGSIZE}" >> ${MYSQL_CONFIG} && \
    echo "innodb_flush_log_at_trx_commit = ${MYSQL_FLUSHLOG}" >> ${MYSQL_CONFIG} && \
    echo "innodb_flush_method = ${MYSQL_FLUSHMETHOD}" >> ${MYSQL_CONFIG}


# this will create DB's on container start, and may take some time depending on your ENV's above
# if you set them inline with -e during docker run, you can override defaults above before db is created
RUN echo 'sysbench --tables=${SYSBENCH_TABLE_COUNT} --table_size=${SYSBENCH_TABLE_SIZE} --mysql-db=${MYSQL_DATABASE} --mysql-user=root oltp_read_write prepare' > /docker-entrypoint-initdb.d/sysbench_prepare.sh && \
    chmod +x /docker-entrypoint-initdb.d/sysbench_prepare.sh


# run /opt/sysbench_run.sh to start a single test run
# if you set them inline with -e during docker exec, you can override defaults above before testing starts
RUN echo 'sysbench --tables=${SYSBENCH_TABLE_COUNT} --table_size=${SYSBENCH_TABLE_SIZE} --time=${SYSBENCH_MAX_TIME} --events=${SYSBENCH_MAX_REQUESTS} --mysql-db=${MYSQL_DATABASE} --mysql-user=root oltp_read_write run' > /opt/sysbench_run.sh && \
    chmod +x /opt/sysbench_run.sh

RUN mkdir /code
COPY ./sysbench-app /code
COPY ./rmqserver /code/rmqserver
COPY ./scripts/init.sh /code/init.sh
RUN chmod 755 /code/init.sh
RUN chmod 755 /code/lin.sh
RUN pip3 install -r /code/rmqserver/requirements.txt
RUN pip3 install -r /code/requirements.txt

WORKDIR /code/

EXPOSE 3306 33060

# CMD ["/code/init.sh"]

CMD /bin/sh /code/init.sh ${HOSTNAME} ${postfix}
