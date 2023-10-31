Thanos provides long-term data storage capability and High availability to our Prometheus. 
 
# Thanos Setup

### Environment

- We will be creating three prometheus containers assuming them as three seprate environment. 

      prometheus-alpha    --> running on port 9091
      prometheus-beta     --> running on port 9092
      prometheus-gamma    --> running on port 9093

- One S3 bucket which will be used as long term data storage. 

      S3 authentication will be happening using the node IAM Role profile.


- Thanos sidecar containers which will be running along with prometheus containers to collect the metrics and push them to S3 bucket.

- One Querier Dashbaord, where we will see all three container 's metrics.

- Thanos Storage Gateway to share all the metrics from all three environments with Thanos as a single unit.


</br>
    
### Generate dummy data for testing
--------------------------------

1. One test file with some data block.

        CURR_DIR=$(pwd)

        docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --max-time=6h > ${CURR_DIR}/block-spec.yaml


2. Data for alpha prometheus


        mkdir ${CURR_DIR}/prom-alpha 

        docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="alpha"' --max-time=6h | docker run -v ${CURR_DIR}/prom-alpha:/out -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir /out


3. Data for beta prometheus

        mkdir ${CURR_DIR}/prom-beta

        docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="beta"' --max-time=6h | docker run -v ${CURR_DIR}/prom-beta:/out -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir /out


4. Data for gamma prometheus

        mkdir ${CURR_DIR}/prom-gamma 

        docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="gamma"' --max-time=6h | docker run -v ${CURR_DIR}/prom-gamma:/out -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir /out


</br>

### Creating prometheus config file for each instance
-------------------------------------------------------

1. For alpha instance

        cat <<EOF > ${CURR_DIR}/prom-alpha-config.yaml
        global:
          scrape_interval: 5s
          external_labels:
            cluster: alpha
            replica: 0
            tenant: team-alpha

        scrape_configs:
          - job_name: 'prometheus'
            static_configs:
              - targets: ['127.0.0.1:9091']
        EOF

2. For beta instance 

        cat <<EOF > ${CURR_DIR}/prom-beta-config.yaml
        global:
          scrape_interval: 5s
          external_labels:
            cluster: beta
            replica: 0
            tenant: team-beta 

        scrape_configs:
          - job_name: 'prometheus'
            static_configs:
              - targets: ['127.0.0.1:9092']
        EOF

3. For gamma instance

        cat <<EOF > ${CURR_DIR}/prom-gamma-config.yaml
        global:
          scrape_interval: 5s
          external_labels:
            cluster: gamma
            replica: 0
            tenant: team-gamma

        scrape_configs:
          - job_name: 'prometheus'
            static_configs:
              - targets: ['127.0.0.1:9093']
        EOF

</br>

### Deploying prometheus containers
--------------------------------------

1. For alpha container


        docker run -d -p 9091:9091 --rm -v ${CURR_DIR}/prom-alpha-config.yaml:/etc/prometheus/prometheus.yml -v ${CURR_DIR}/prom-alpha:/prometheus -u root --name prom-alpha quay.io/prometheus/prometheus:v2.20.0 \
            --config.file=/etc/prometheus/prometheus.yml \
            --storage.tsdb.path=/prometheus \
            --storage.tsdb.retention.time=1000d \
            --storage.tsdb.max-block-duration=2h \
            --storage.tsdb.min-block-duration=2h \
            --web.listen-address=:9091 \
            --web.external-url=http://localhost:9091 \
            --web.enable-lifecycle \
            --web.enable-admin-api



2. For beta container


        docker run -d -p 9092:9092 --rm -v ${CURR_DIR}/prom-beta-config.yaml:/etc/prometheus/prometheus.yml -v ${CURR_DIR}/prom-beta:/prometheus -u root --name prom-beta quay.io/prometheus/prometheus:v2.20.0 \
            --config.file=/etc/prometheus/prometheus.yml \
            --storage.tsdb.path=/prometheus \
            --storage.tsdb.retention.time=1000d \
            --storage.tsdb.max-block-duration=2h \
            --storage.tsdb.min-block-duration=2h \
            --web.listen-address=:9092 \
            --web.external-url=http://localhost:9092 \
            --web.enable-lifecycle \
            --web.enable-admin-api


3. For gamma container


        docker run -d -p 9093:9093 --rm -v ${CURR_DIR}/prom-gamma-config.yaml:/etc/prometheus/prometheus.yml -v ${CURR_DIR}/prom-gamma:/prometheus -u root --name prom-gamma quay.io/prometheus/prometheus:v2.20.0 \
            --config.file=/etc/prometheus/prometheus.yml \
            --storage.tsdb.path=/prometheus \
            --storage.tsdb.retention.time=1000d \
            --storage.tsdb.max-block-duration=2h \
            --storage.tsdb.min-block-duration=2h \
            --web.listen-address=:9093 \
            --web.external-url=http://localhost:9093 \
            --web.enable-lifecycle \
            --web.enable-admin-api


</br>

### Deploying Sidecar container
--------------------------------

1. For alpha container

        docker run -d -p 19191:19191 --rm \
            -v ${CURR_DIR}/prom-alpha-config.yaml:/etc/prometheus/prometheus.yml \
            --link prom-alpha:prom-alpha \
            --name prom-alpha-sidecar \
            -u root \
            quay.io/thanos/thanos:v0.26.0 \
            sidecar \
            --http-address 0.0.0.0:19091 \
            --grpc-address 0.0.0.0:19191 \
            --reloader.config-file /etc/prometheus/prometheus.yml \
            --prometheus.url "http://prom-alpha:9091"


2. For beta container

        docker run -d -p 19192:19192 --rm \
            -v ${CURR_DIR}/prom-beta-config.yaml:/etc/prometheus/prometheus.yml \
            --link prom-beta:prom-beta \
            --name prom-beta-sidecar \
            -u root \
            quay.io/thanos/thanos:v0.26.0 \
            sidecar \
            --http-address 0.0.0.0:19092 \
            --grpc-address 0.0.0.0:19192 \
            --reloader.config-file /etc/prometheus/prometheus.yml \
            --prometheus.url "http://prom-beta:9092"


3. For gamma container


        docker run -d -p 19193:19193 --rm \
            -v ${CURR_DIR}/prom-gamma-config.yaml:/etc/prometheus/prometheus.yml \
            --link prom-gamma:prom-gamma \
            --name prom-gamma-sidecar \
            -u root \
            quay.io/thanos/thanos:v0.26.0 \
            sidecar \
            --http-address 0.0.0.0:19093 \
            --grpc-address 0.0.0.0:19193 \
            --reloader.config-file /etc/prometheus/prometheus.yml \
            --prometheus.url "http://prom-gamma:9093"


</br>

### Global View / HA Querier
----------------------------


      docker run -d -p 9090:9090 --rm \
          --name querier \
          --link prom-alpha-sidecar:prom-alpha-sidecar \
          --link prom-beta-sidecar:prom-beta-sidecar \
          --link prom-gamma-sidecar:prom-gamma-sidecar \
          quay.io/thanos/thanos:v0.26.0 \
          query \
          --http-address 0.0.0.0:9090 \
          --grpc-address 0.0.0.0:19190 \
          --store prom-alpha-sidecar:19191 \
          --store prom-beta-sidecar:19192 \
          --store prom-gamma-sidecar:19193


</br>

### S3 Bucket and IAM Role
---------------------------

1. Create one s3 bucket in your aws account.
2. Create one IAM Role with PutObject permission to upload the data into this bucket.
3. Attach this IAM role to the Virtual Machine. (VM where we are running all these containers)

</br>

### Create the storage config file with s3 details
---------------------------------------------------

      vi storage-config.yaml

      type: S3
      config:
        bucket: "vivekjain-thanos-test"
        signature_version2: false
        endpoint: "s3.ap-south-1.amazonaws.com"

</br>

### Recreate the sidecar container with storage-config
------------------------------------------------------

Recreation is required so that the thanos sidecar container can connect to s3 and push data blocks.

1. For alpha container


        docker run -d -p 19191:19191 --rm \
            -v ${CURR_DIR}/prom-alpha-config.yaml:/etc/prometheus/prometheus.yml \
            -v ${CURR_DIR}/storage-config.yaml:/etc/thanos/storage-config.yaml \
            -v ${CURR_DIR}/prom-alpha:/prometheus \
            --link prom-alpha:prom-alpha \
            --name prom-alpha-sidecar \
            -u root \
            quay.io/thanos/thanos:v0.26.0 \
            sidecar \
            --tsdb.path /prometheus \
            --objstore.config-file /etc/thanos/storage-config.yaml \
            --shipper.upload-compacted \
            --http-address 0.0.0.0:19091 \
            --grpc-address 0.0.0.0:19191 \
            --reloader.config-file /etc/prometheus/prometheus.yml \
            --prometheus.url "http://prom-alpha:9091"


2. For beta container


        docker run -d -p 19192:19192 --rm \
            -v ${CURR_DIR}/prom-beta-config.yaml:/etc/prometheus/prometheus.yml \
            -v ${CURR_DIR}/storage-config.yaml:/etc/thanos/storage-config.yaml \
            -v ${CURR_DIR}/prom-beta:/prometheus \
            --link prom-beta:prom-beta \
            --name prom-beta-sidecar \
            -u root \
            quay.io/thanos/thanos:v0.26.0 \
            sidecar \
            --tsdb.path /prometheus \
            --objstore.config-file /etc/thanos/storage-config.yaml \
            --shipper.upload-compacted \
            --http-address 0.0.0.0:19092 \
            --grpc-address 0.0.0.0:19192 \
            --reloader.config-file /etc/prometheus/prometheus.yml \
            --prometheus.url "http://prom-beta:9092"


3. For gamma container


        docker run -d -p 19193:19193 --rm \
            -v ${CURR_DIR}/prom-gamma-config.yaml:/etc/prometheus/prometheus.yml \
            -v ${CURR_DIR}/storage-config.yaml:/etc/thanos/storage-config.yaml \
            -v ${CURR_DIR}/prom-gamma:/prometheus \
            --link prom-gamma:prom-gamma \
            --name prom-gamma-sidecar \
            -u root \
            quay.io/thanos/thanos:v0.26.0 \
            sidecar \
            --tsdb.path /prometheus \
            --objstore.config-file /etc/thanos/storage-config.yaml \
            --shipper.upload-compacted \
            --http-address 0.0.0.0:19093 \
            --grpc-address 0.0.0.0:19193 \
            --reloader.config-file /etc/prometheus/prometheus.yml \
            --prometheus.url "http://prom-gamma:9093"

</br>

### Create the storage api container
-------------------------------------

A single api to access all the data from s3 bucket (data from all three instances)

      docker run -d -p 19194:19194 --rm \
          -v ${CURR_DIR}/storage-config.yaml:/etc/thanos/storage-config.yaml \
          --name store-gateway \
          quay.io/thanos/thanos:v0.26.0 \
          store \
          --objstore.config-file /etc/thanos/storage-config.yaml \
          --http-address 0.0.0.0:19094 \
          --grpc-address 0.0.0.0:19194    


</br>

### Recreate the Querier with Storage API
-----------------------------------------


      docker run -d -p 9090:9090 --rm \
          --name querier \
          --link prom-alpha-sidecar:prom-alpha-sidecar \
          --link prom-beta-sidecar:prom-beta-sidecar \
          --link prom-gamma-sidecar:prom-gamma-sidecar \
          --link store-gateway:store-gateway \
          quay.io/thanos/thanos:v0.26.0 \
          query \
          --http-address 0.0.0.0:9090 \
          --grpc-address 0.0.0.0:19190 \
          --store prom-alpha-sidecar:19191 \
          --store prom-beta-sidecar:19192 \
          --store prom-gamma-sidecar:19193 \
          --store store-gateway:19194
          
</br>

#### Referenece and Acknowledgement
------------------------------------

- https://thanos.io/
- Blogs and Videos released by Thanos community.
- Video session of Bartek Plotka with Rawkode.
