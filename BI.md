# Building Elasticsearch

The instructions provided below specify the steps to build [Elasticsearch](https://www.elastic.co/products/elasticsearch) 7.17.22 on Linux on IBM Z for following distributions:

* RHEL (9.2, 9.4)

_**General Notes**_

* _When following the steps below please use a standard permission user unless otherwise specified._
* _A directory `/<source_root>/` will be referred to in these instructions, this is a temporary writable directory anywhere you'd like to place it._

## Build and Install Elasticsearch

### 1) Build using script

If you want to build Elasticsearch using manual steps, go to STEP 2.

Use the following commands to build Elasticsearch using the build [script](https://github.com/linux-on-ibm-z/scripts/tree/master/Elasticsearch). Please make sure you have `wget` installed.

```bash
wget -q https://raw.githubusercontent.com/linux-on-ibm-z/scripts/master/Elasticsearch/7.17.22/build_elasticsearch.sh

# Build Elasticsearch
bash build_elasticsearch.sh  [Provide -t option for executing build with tests]
```

If the build completes successfully, go to STEP 8. In case of error, check `logs` for more details or go to STEP 2 to follow manual build steps.

### 2) Install build dependencies

```bash
export SOURCE_ROOT=/<source_root>/
```

* RHEL (9.2, 9.4)

  ```bash
  sudo yum install -y curl git gzip tar wget patch
  ```

  * Download and install Eclipse Adoptium Temurin Runtime (Java 17) from [here](https://adoptium.net/temurin/releases/?version=17).

_**Note:** At the time of creation of these build instructions, Elasticsearch was verified with Eclipse Adoptium Temurin Runtime (Java 17) version (build 17.0.13+11)._

### 3) Set the environment variables

```bash
export LANG="en_US.UTF-8"
export JAVA_HOME=<Path to JDK>
export ES_JAVA_HOME=/<Path to JDK>/
export JAVA17_HOME=/<Path to JDK>/
export PATH=$ES_JAVA_HOME/bin:$PATH
```

_**Note:** Ensure system locale is set up correctly for Elasticsearch to build without encoding errors._

### 4) Download Elasticsearch and apply patches

```bash
cd $SOURCE_ROOT
git clone https://github.com/elastic/elasticsearch
cd elasticsearch
git checkout v7.17.22
```

* Apply gradle patches to create s390x distribution

```bash
 export PATCH_URL="https://raw.githubusercontent.com/linux-on-ibm-z/scripts/master/Elasticsearch/7.17.22/patch/elasticsearch.patch"
 curl -o elasticsearch.patch $PATCH_URL
 git apply elasticsearch.patch
```

### 5) Build

```
cd $SOURCE_ROOT/elasticsearch
CPU_NUM="$(grep -c ^processor /proc/cpuinfo)"
./gradlew :distribution:archives:oss-linux-s390x-tar:assemble --max-workers="$CPU_NUM"  --parallel
```

### 6) Testing (Optional)

```bash
cd $SOURCE_ROOT/elasticsearch
export JAVA_TOOL_OPTIONS="-Dfile.encoding=UTF8"
export JAVA_HOME=/<Path to JDK>/
export RUNTIME_JAVA_HOME=/<Path to JDK>/
./gradlew --continue test -Dtests.haltonfailure=false -Dtests.jvm.argline="-Xss2m"
```

_**Notes:**_

* _You can set `RUNTIME_JAVA_HOME` optionally to the location of the JDK (another version of the JDK that act as the runtime) that you'd like to use during testing. Eclipse Temurin 17 (previously known as AdoptOpenJDK with Hotspot) was used at the time of this writing._
* _Certain test cases may require an individual rerun to pass. There may be false negatives due to seccomp not supporting s390x properly._
* _If there is an stack overflow error, increase the `-Xss` arg value in the above command._
* _Some X-Pack test cases will fail as X-Pack plugins are not supported on s390x, such as Machine Learning features._
* _The test case failures in modules `server:test` can be ignored. They passed on re-run._
* _The node.processors setting is now bounded by the number of available processors. Some X-Pack Test case may fail for this reason. To fix this, ensure the value of node.processors setting does not exceed the number of available processors._
* _`Illegal reflective access` warnings in the logs are known issues. See https://github.com/elastic/elasticsearch/issues/40334._
* _For more information regarding Elasticsearch testing, please refer to their testing [documentation](https://github.com/elastic/elasticsearch/blob/v7.17.22/TESTING.asciidoc)._
* _User can also create distributions as deb, rpm and docker using below commands. In case of docker distribution, User needs to change Dockerfile to use built tar distribution instead of downloading it._

```bash
./gradlew :distribution:packages:s390x-oss-deb:assemble
./gradlew :distribution:packages:s390x-oss-rpm:assemble
./gradlew :distribution:docker:oss-docker-s390x-build-context:assemble
```

### 7) Install Elasticsearch

```bash
cd $SOURCE_ROOT/elasticsearch
sudo mkdir /usr/share/elasticsearch
sudo tar -xzf distribution/archives/oss-linux-s390x-tar/build/distributions/elasticsearch-oss-7.17.22-SNAPSHOT-linux-s390x.tar.gz -C /usr/share/elasticsearch --strip-components 1
sudo ln -sf /usr/share/elasticsearch/bin/* /usr/bin/

sudo /usr/sbin/groupadd elastic
sudo chown [username]:elastic -R /usr/share/elasticsearch/
```

* Update configurations to disable unsupported xpack.ml
```bash
sudo echo 'xpack.ml.enabled: false' >> /usr/share/elasticsearch/config/elasticsearch.yml
```

### 8) Verify Elasticsearch Server

```bash
> elasticsearch --version
Version: 7.17.22-SNAPSHOT, Build: oss/tar/38e9ca2e81304a821c50862dafab089ca863944b/2024-10-27T21:02:23.220279474Z, JVM: 17.0.13
```

### 9) Start Elasticsearch Server

```bash
elasticsearch &
```

Use `curl -X GET http://localhost:9200/` when running Elasticsearch. The output should be similar to this:
```
{
  "name" : "055f3cbd8e0c",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "3Go4-W6ASBO09fN8km2avQ",
  "version" : {
    "number" : "7.17.22-SNAPSHOT",
    "build_flavor" : "oss",
    "build_type" : "tar",
    "build_hash" : "38e9ca2e81304a821c50862dafab089ca863944b",
    "build_date" : "2024-10-27T21:02:23.220279474Z",
    "build_snapshot" : true,
    "lucene_version" : "8.11.3",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

```

_**Note:**_

* _If Access denied to Keystore error occurs on starting Elasticsearch, try updating/creating new password for the keystore using_
```bash
elasticsearch-keystore create -p
```

## Installing Elasticsearch Curator client (Optional)

Elasticsearch Curator can be used to curate, or manage, Elasticsearch indices and snapshots. It can be installed using below steps:

### 1) Install dependencies

* RHEL (9.2, 9.4)

  ```bash
  sudo yum install -y python3-devel libyaml-devel
  ```
### 2) Install Elasticsearch Curator

```bash
  sudo pip3 install elasticsearch-curator
```
_**Note:**_

* _In case of an error `sudo: pip3: command not found`, Run above command as `sudo env PATH=$PATH -H pip3 install elasticsearch-curator`._

### 3) Verify Elasticsearch Curator installation

```bash
> curator --version
curator, version 8.0.17
```
_**Note:**_

* _In case an error related to ASCII encoding is thrown by Click, ensure your locale is correctly set._

## References:
- <https://www.elastic.co/products/elasticsearch>
- <https://www.elastic.co/guide/en/elasticsearch/client/curator/current/index.html>
