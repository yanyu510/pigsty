# Configuration Guide



## TL;DR

* Ansible style. refer to ansible variable precedence.
* Just change parameters you need

* Read detail [document](../roles/)



## Example

* [`group_vars/all.yml`](../group_vars/all.yml`) defines most infrastructure parameters and global postgres cluster parameters.
* [`cls/inventory.ini`](../inventory/inventory.ini) defines postgres cluster and override parameters



## Explain

```yaml
---
#------------------------------------------------------------------------------
# Pigsty configuration file
#------------------------------------------------------------------------------
#
# This file using yaml format
#
# This file is used as default settings for all cluster. it will override role's
# default variable, and will be override by host inventory variables
#
#------------------------------------------------------------------------------


#------------------------------------------------------------------------------
# CONNECTION PARAMETERS
#------------------------------------------------------------------------------
# this section defines connection parameters

# ansible_user: vagrant             # admin user with ssh access and sudo privilege

proxy_env:                          # global proxy env when downloading packages
  no_proxy: "localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,*.pigsty,*.aliyun.com"

#------------------------------------------------------------------------------
# REPO PROVISION
#------------------------------------------------------------------------------
# this section defines how to build a local repo

repo_enabled: true                            # build local yum repo on meta nodes?
repo_name: pigsty                             # local repo name
repo_address: yum.pigsty                      # repo external address (ip:port or url)
repo_port: 80                                 # listen address, must same as repo_address
repo_home: /www                               # default repo dir location
repo_rebuild: false                           # force re-download packages
repo_remove: true                             # remove existing repos
# repo_upstreams: []                          # use role default upstream
# repo_packages: []                           # use role default packages
# repo_url_packages: []                       # use role default web urls


#------------------------------------------------------------------------------
# NODE PROVISION
#------------------------------------------------------------------------------
# this section defines how to provision nodes

# - node dns - #
node_dns_hosts:                               # static dns records in /etc/hosts
  - 10.10.10.10 yum.pigsty
node_dns_server: add                          # add (default) | none (skip) | overwrite (remove old settings)
node_dns_servers:                             # dynamic nameserver in /etc/resolv.conf
  - 10.10.10.10
node_dns_options:                             # dns resolv options
  - options single-request-reopen timeout:1 rotate
  - domain service.consul

# - node repo - #
node_repo_method: local                       # none|local|public (use local repo for production env)
node_repo_remove: true                        # whether remove existing repo
node_local_repo_url:                          # local repo url (if method=local)
  - http://yum.pigsty/pigsty.repo

# - node packages - #
node_extra_packages:                          # extra packages for all nodes (install postgres12 to accelerate initdb)
  - postgresql12*,postgis30_12*,timescaledb_12,citus_12,pglogical_12   # postgres 12 basic
  - pg_qualstats12,pg_cron_12,pg_top12,pg_repack12,pg_squeeze12,pg_stat_kcache12,wal2json12
  - python36-requests,python3-consul,pgbouncer,patroni,pg_exporter,pg_top,pgbadger

# node_packages: []                           # common packages for all nodes (use role default)
# node_meta_packages: []                      # packages for meta nodes only (use role default)

# - node features - #
node_disable_numa: false                      # disable numa, important for production database, reboot required
node_disable_swap: true                       # disable swap, important for production database
node_disable_firewall: true                   # disable firewall (required if using kubernetes)
node_disable_selinux: true                    # disable selinux  (required if using kubernetes)
node_static_network: true                     # keep dns resolver settings after reboot
node_disk_prefetch: false                     # setup disk prefetch on HDD to increase performance

# - node kernel modules - #
node_kernel_modules: [softdog, br_netfilter, ip_vs, ip_vs_rr, ip_vs_rr, ip_vs_wrr, ip_vs_sh, nf_conntrack_ipv4]

# - node tuned - #
node_tune: tiny                               # install and activate tuned profile: none|oltp|olap|crit|tiny
node_sysctl_params:                           # set additional sysctl parameters, k:v format
  net.bridge.bridge-nf-call-iptables: 1       # for kubernetes

# - node user - #
node_admin_setup: true                        # setup an default admin user ?
node_admin_uid: 88                            # uid and gid for admin user
node_admin_username: admin                    # default admin user
node_admin_ssh_exchange: true                 # exchange ssh key among cluster ?
node_admin_pks: []                            # public key list that will be installed

# - node ntp - #
node_ntp_service: ntp                         # ntp or chrony
node_ntp_config: true                         # overwrite existing ntp config?
node_timezone: Asia/Shanghai                  # default node timezone
node_ntp_servers:                             # default NTP servers
  - pool cn.pool.ntp.org iburst
  - pool pool.ntp.org iburst
  - pool time.pool.aliyun.com iburst
  - server 10.10.10.10 iburst


#------------------------------------------------------------------------------
# META PROVISION
#------------------------------------------------------------------------------
# - ca - #
ca_method: create                             # create|copy|recreate
ca_subject: "/CN=root-ca"                     # self-signed CA subject
ca_homedir: /ca                               # ca cert directory
ca_cert: ca.crt                               # ca public key/cert
ca_key: ca.key                                # ca private key

# - nginx - #
nginx_upstream:
  - {name: consul,        host: c.pigsty, url: "127.0.0.1:8500"}
  - {name: grafana,       host: g.pigsty, url: "127.0.0.1:3000"}
  - {name: prometheus,    host: p.pigsty, url: "127.0.0.1:9090"}
  - {name: alertmanager,  host: a.pigsty, url: "127.0.0.1:9093"}

# - nameserver - #
dns_records:                                  # dynamic dns record resolved by dnsmasq
  - 10.10.10.2  pg-meta                       # sandbox vip for pg-meta
  - 10.10.10.3  pg-test                       # sandbox vip for pg-test
  - 10.10.10.10 meta-1                        # sandbox node meta-1 (node-0)
  - 10.10.10.11 node-1                        # sandbox node node-1
  - 10.10.10.12 node-2                        # sandbox node node-2
  - 10.10.10.13 node-3                        # sandbox node node-3
  - 10.10.10.10 pigsty
  - 10.10.10.10 y.pigsty yum.pigsty
  - 10.10.10.10 c.pigsty consul.pigsty
  - 10.10.10.10 g.pigsty grafana.pigsty
  - 10.10.10.10 p.pigsty prometheus.pigsty
  - 10.10.10.10 a.pigsty alertmanager.pigsty
  - 10.10.10.10 n.pigsty ntp.pigsty

# - prometheus - #
prometheus_scrape_interval: 2s                # global scrape & evaluation interval (2s for dev, 15s for prod)
prometheus_scrape_timeout: 1s                 # global scrape timeout (1s for dev, 8s for prod)
prometheus_metrics_path: /metrics             # default metrics path (only affect job 'pg')
prometheus_data_dir: /export/prometheus/data  # prometheus data dir
prometheus_retention: 30d                     # how long to keep

# - grafana - #
grafana_url: http://10.10.10.10:3000           # grafana url
grafana_admin_password: admin                  # default grafana admin user password
grafana_plugin: install                        # none|install|reinstall
grafana_cache: /www/pigsty/grafana/plugins.tar.gz # path to grafana plugins tarball
grafana_provision_mode: db                     # none|db|api
# grafana_plugins: []                          # grafana plugins list (use role default)
# grafana_git_plugins: []                      # grafana plugins from git (use role default)
# grafana_dashboards: []                       # default dashboards (use role default)


#------------------------------------------------------------------------------
# DCS PROVISION
#------------------------------------------------------------------------------
dcs_type: consul                              # consul | etcd | both
dcs_name: pigsty                              # consul dc name | etcd initial cluster token
dcs_servers:                                  # dcs server dict in name:ip format
  meta-1: 10.10.10.10                         # you could use existing dcs cluster
  # meta-2: 10.10.10.11                       # host which have their IP listed here will be init as server
  # meta-3: 10.10.10.12                       # 3 or 5 dcs nodes are recommend for production environment

dcs_exists_action: skip                       # abort|skip|clean if dcs server already exists
consul_data_dir: /var/lib/consul              # consul data dir (/var/lib/consul by default)
etcd_data_dir: /var/lib/etcd                  # etcd data dir (/var/lib/consul by default)


#------------------------------------------------------------------------------
# POSTGRES INSTALLATION
#------------------------------------------------------------------------------
# - dbsu - #
pg_dbsu: postgres                             # os user for database, postgres by default (change it is not recommended!)
pg_dbsu_uid: 26                               # os dbsu uid and gid, 26 for default postgres users and groups
pg_dbsu_sudo: limit                           # none|limit|all|nopass (Privilege for dbsu, limit is recommended)
pg_dbsu_home: /var/lib/pgsql                  # postgresql binary
pg_dbsu_ssh_exchange: false                   # exchange ssh key among same cluster

# - postgres packages - #
pg_version: 12                                # default postgresql version
pgdg_repo: false                              # use official pgdg yum repo (disable if you have local mirror)
pg_add_repo: false                            # add postgres related repo before install (useful if you want a simple install)
pg_bin_dir: /usr/pgsql/bin                    # postgres binary dir
#pg_packages:                                 # packages to be installed  (comment this will use role default)
#  - postgresql${pg_version}*
#  - postgis30_${pg_version}* timescaledb_${pg_version} citus_${pg_version} pglogical_${pg_version}   # postgres ${pg_version} basic
#  - pg_qualstats${pg_version} pg_cron_${pg_version} pg_top${pg_version} pg_repack${pg_version} pg_squeeze${pg_version} pg_stat_kcache${pg_version} wal2json${pg_version}
#  - pgpool-II-${pg_version} pgpool-II-${pg_version}-extensions
#  - pgbouncer patroni pg_exporter pg_top pgbadger # postgres common utils

#pg_packages:                                 # use this packages list instead if pg_version=13
#  - postgresql${pg_version}*
#  - pgbouncer patroni pg_exporter pg_top pgbadger

# - 3rd party extensions #
# pg_extensions: []                           # available extensions (comment this will use role default)
#  - ddlx_${pg_version} bgw_replstatus${pg_version} count_distinct${pg_version} extra_window_functions_${pg_version}
#  - geoip${pg_version} hll_${pg_version} hypopg_${pg_version} ip4r${pg_version} jsquery_${pg_version} multicorn${pg_version}
#  - osm_fdw${pg_version} mysql_fdw_${pg_version} ogr_fdw${pg_version} mongo_fdw${pg_version} hdfs_fdw_${pg_version} cstore_fdw_${pg_version}
#  - wal2mongo${pg_version} orafce${pg_version} pagila${pg_version} pam-pgsql${pg_version} passwordcheck_cracklib${pg_version} periods_${pg_version}
#  - pg_auto_failover_${pg_version} pg_bulkload${pg_version} pg_catcheck${pg_version} pg_comparator${pg_version} pg_filedump${pg_version}
#  - pg_fkpart${pg_version} pg_jobmon${pg_version} pg_partman${pg_version} pg_pathman${pg_version} pg_track_settings${pg_version}
#  - pg_wait_sampling_${pg_version} pgagent_${pg_version} pgaudit14_${pg_version} pgauditlogtofile-${pg_version} pgbconsole${pg_version}
#  - pgcryptokey${pg_version} pgexportdoc${pg_version} pgfincore${pg_version} pgimportdoc${pg_version} pgmemcache-${pg_version} pgmp${pg_version}
#  - pgq-${pg_version} pgrouting_${pg_version} pgtap${pg_version} plpgsql_check_${pg_version} plr${pg_version} plsh${pg_version}
#  - postgresql_anonymizer${pg_version} postgresql-unit${pg_version} powa_${pg_version} prefix${pg_version} repmgr${pg_version}
#  - safeupdate_${pg_version} semver${pg_version} slony1-${pg_version} sqlite_fdw${pg_version} sslutils_${pg_version} system_stats_${pg_version}
#  - table_version${pg_version} topn_${pg_version}


#------------------------------------------------------------------------------
# POSTGRES CLUSTER PROVISION
#------------------------------------------------------------------------------
# - identity - #
# pg_cluster:                                 # [REQUIRED] cluster name (validated during pg_preflight)
# pg_seq: 0                                   # [REQUIRED] instance seq (validated during pg_preflight)
# pg_role: replica                            # [REQUIRED] service role (validated during pg_preflight)
# pg_hostname: false                          # overwrite node hostname with pg instance name

# - retention - #
# pg_exists_action, available options: abort|clean|skip
#  - abort: abort entire play's execution (default)
#  - clean: remove existing cluster (dangerous)
#  - skip: end current play for this host
# pg_exists: false                              # auxiliary flag variable (DO NOT SET THIS)
# pg_exists_action: abort

# - storage - #
# pg_data: /pg/data                             # postgres data directory
# pg_fs_main: /export                           # data disk mount point     /pg -> {{ pg_fs_main }}/postgres/{{ pg_instance }}
# pg_fs_bkup: /var/backups                      # backup disk mount point   /pg/* -> {{ pg_fs_bkup }}/postgres/{{ pg_instance }}/*

# - connection - #
# pg_listen: '0.0.0.0'                          # postgres listen address, '0.0.0.0' by default (all ipv4 addr)
# pg_port: 5432                                 # postgres port (5432 by default)

# - patroni - #
# patroni_mode, available options: default|pause|remove
# default: default ha mode
# pause:   into maintainance mode
# remove:  remove patroni after bootstrap
# patroni_mode: default                         # pause|default|remove
# pg_namespace: /pg                             # top level key namespace in dcs
# patroni_port: 8008                            # default patroni port
# patroni_watchdog_mode: automatic              # watchdog mode: off|automatic|required

# - template - #
# pg_conf: patroni.yml                          # user provided patroni config template path
# pg_init: initdb.sh                            # user provided post-init script path, default: initdb.sh

# - authentication - #
pg_hba_common:
  - '"# allow: meta node access with password"'
  - host    all     all                         10.10.10.10/32      md5
  - '"# allow: intranet admin role with password"'
  - host    all     +dbrole_admin               10.0.0.0/8          md5
  - host    all     +dbrole_admin               192.168.0.0/16      md5
  - '"# allow local (pgbouncer) read-write user (production user) password access"'
  - local   all     +dbrole_readwrite                               md5
  - host    all     +dbrole_readwrite           127.0.0.1/32        md5
  - '"# intranet common user password access"'
  - host    all             all                 10.0.0.0/8          md5
  - host    all             all                 192.168.0.0/16      md5
pg_hba_primary: []
pg_hba_replica:
  - '"# allow remote readonly user (stats, personal user) password access (directly)"'
  - local   all     +dbrole_readonly                               md5
  - host    all     +dbrole_readonly           127.0.0.1/32        md5
pg_hba_pgbouncer:
  - '# biz_user intranet password access'
  - local  all          all                                     md5
  - host   all          all                     127.0.0.1/32    md5
  - host   all          all                     10.0.0.0/8      md5
  - host   all          all                     192.168.0.0/16  md5

# - credential - #
# pg_dbsu_password: ''                          # dbsu password (leaving blank will disable sa password login)
# pg_replication_username: replicator           # replication user
# pg_replication_password: replicator           # replication password
# pg_monitor_username: dbuser_monitor           # monitor user
# pg_monitor_password: dbuser_monitor           # monitor password

# - default - #
# pg_default_username: postgres                 # non 'postgres' will create a default admin user (not superuser)
# pg_default_password: postgres                 # dbsu password, omit for 'postgres'
# pg_default_database: postgres                 # non 'postgres' will create a default database
# pg_default_schema: public                     # default schema will be create under default database and used as first element of search_path
# pg_default_extensions: "tablefunc,postgres_fdw,file_fdw,btree_gist,btree_gin,pg_trgm"

# - pgbouncer - #
# pgbouncer_port: 6432                          # default pgbouncer port
# pgbouncer_poolmode: transaction               # default pooling mode: transaction pooling
# pgbouncer_max_db_conn: 100                    # important! do not set this larger than postgres max conn or conn limit


#------------------------------------------------------------------------------
# MONITOR PROVISION
#------------------------------------------------------------------------------
# - monitor options -
node_exporter_port: 9100                # default port for node exporter
pg_exporter_port: 9630                  # default port for pg exporter
pgbouncer_exporter_port: 9631           # default port for pgbouncer exporter
exporter_metrics_path: /metrics         # default metric path for pg related exporter


#------------------------------------------------------------------------------
# PROXY PROVISION
#------------------------------------------------------------------------------
# - vip - #
# vip_enabled: false                            # level2 vip requires primary/standby under same switch
# vip_address: 127.0.0.1                        # virtual ip address ip/cidr
# vip_cidrmask: 32                              # virtual ip address cidr mask
# vip_interface: eth0                           # virtual ip network interface

# - haproxy - #
haproxy_enabled: true                         # enable haproxy among every cluster members
haproxy_policy: leastconn                     # roundrobin, leastconn
haproxy_admin_username: admin                 # default haproxy admin username
haproxy_admin_password: admin                 # default haproxy admin password
haproxy_client_timeout: 3h                    # client side connection timeout
haproxy_server_timeout: 3h                    # server side connection timeout
haproxy_exporter_port: 9101                   # default admin/exporter port
haproxy_check_port: 8008                      # default health check port (patroni 8008 by default)
haproxy_primary_port:  5433                   # default primary port 5433
haproxy_replica_port:  5434                   # default replica port 5434
haproxy_backend_port:  6432                   # default target port: pgbouncer:6432 postgres:5432

...
```







## 配置详情

项目的配置文件分为四部分：

**全局变量定义**

全局变量默认定义于`group_vars/all.yml`，针对不同的环境（开发，测试，生产），可以使用不同的全局变量，并通过软连接将`all.yml`指向对应的环境配置。
全局变量针对所有机器生效，当用户希望使用统一的配置时，例如在所有机器上配置相同的 DNS，NTP Server，安装相同的软件包，使用统一的su密码时，可以修改全局变量
全局变量定义分为8个部分，具体的配置项请参阅文档

* 连接信息
* 本地源定义
* 机器节点初始化
* 控制节点初始化
* DCS元数据库初始化
* Postgres安装
* Postgres集群初始化
* 监控初始化
* 负载均衡代理初始化

**主机变量定义**

主机清单（IP，ssh信息，主机变量）默认定义于`cls/inventory.yml`，该文件包含了一套环境中所有主机相关的信息。可以通过`ansible -i <path>`使用其他的主机清单文件。
主机清单使用`ini`格式，定义了一系列分组，默认分组`meta`包含了控制节点的信息。其他分组每个都包含了一个数据库集群的定义。
例如，下面的例子定义了一个名为`pg-test`的集群，其中有三个实例，`10.10.10.11`为主库，`10.10.10.12`与`10.10.10.13`为从库，安装12版本的PostgreSQL数据库。

```ini
[pg-test]
10.10.10.11 ansible_host=node-1 pg_role=primary pg_seq=1
10.10.10.12 ansible_host=node-2 pg_role=replica pg_seq=2
10.10.10.13 ansible_host=node-3 pg_role=replica pg_seq=3

[pg-test:vars]
pg_cluster = pg-test
pg_version = 12
```

**数据库初始化模板**

初始化模板是用于初始化数据库集群的定义文件，默认位于`roles/postgres/templates/patroni.yml`，采用`patroni.yml` [配置文件格式](https://patroni.readthedocs.io/en/latest/SETTINGS.html)
在[`templates/`](templates/)目录中，有四种预定义好的初始化模板：

* [`oltp.yml`](oltp.yml) 常规OLTP模板，默认配置
* [`olap.yml`](olap.yml) OLAP模板，提高并行度，针对吞吐量优化，针对长时间运行的查询进行优化。
* [`crit.yml`](crit.yml) 核心业务模板，基于OLTP模板针对安全性，数据完整性进行优化，采用同步复制，启用数据校验和。
* [`tiny.yml`](tiny.yml) 微型数据库模板，针对低资源场景进行优化，例如运行于虚拟机中的演示数据库集群。

用户也可以基于上述模板进行定制与修改，并通过`pg_conf`参数使用相应的模板。


**数据库初始化脚本**

当数据库初始化完毕后，用户通常希望对数据库进行自定义的定制脚本，例如创建统一的默认角色，用户，创建默认的模式，配置默认权限等。
本项目提供了一个默认的初始化脚本`roles/postgres/templates/initdb.sh`，基于以下几个变量创建默认的数据库与用户。

```yaml
pg_default_username: postgres                 # non 'postgres' will create a default admin user (not superuser)
pg_default_password: postgres                 # dbsu password, omit for 'postgres'
pg_default_database: postgres                 # non 'postgres' will create a default database
pg_default_schema: public                     # default schema will be create under default database and used as first element of search_path
pg_default_extensions: "tablefunc,postgres_fdw,file_fdw,btree_gist,btree_gin,pg_trgm"
```

用户可以基于本脚本进行定制，并通过`pg_init`参数使用相应的自定义脚本。
