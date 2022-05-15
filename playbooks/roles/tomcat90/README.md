# Ansible Role - tomcat8

## java
---

### OpenJDK vs OracleJDK
- OracleJDK의 라이센스 구비하고 있지 않으므로 사용 하지 않음
- OpenJDK는 GPLv2 라이센스이며 기업에서 상업용 목적으로 사용이 가능함.
- 성능면에서 큰 차이 없음.
- Oracle JDK = OpenJDK + 상업용 코드
> Q: What is the difference between the source code found in the OpenJDK repository, and the code you use to build the Oracle JDK?
> A: It is very close - our build process for Oracle JDK releases builds on OpenJDK 7 by adding just a couple of pieces, like the deployment code, which includes Oracle's implementation of the Java Plugin and Java WebStart, as well as some closed source third party components like a graphics rasterizer, some open source third party components, like Rhino, and a few bits and pieces here and there, like additional documentation or third party fonts. Moving forward, our intent is to open source all pieces of the Oracle JDK except those that we consider commercial features such as JRockit Mission Control (not yet available in Oracle JDK), and replace encumbered third party components with open source alternatives to achieve closer parity between the code bases.

### JDK vs JRE and headless
- JDK : JRE + Development Tool
- JRE : JVM + Library Classes
- headless : Minimal Java runtime - needed for executing non GUI Java programs, using Hotspot JIT.
> GUI를 쓰지 않고, 개발서버에서 직접 컴파일, 디버깅등의 개발 관련 업무를 하지 않는 다면 headless 가 적합

### 8 or 9 or 10 or 11
- JAVA에도 LTS 버전이 존재 하며 8과 11이 LTS 버전이다.
- 본 role에서는 `java_version` 변수를 통해 8, 11 버전 지정이 가능하다.

## GC
- G1GC 로 고정 설정

---

## tomcat
- ubuntu 16.04에서는 `tomcat8.5` 버전의 바이너리 설치를 지원하지 않아 소스 설치 하였음.
- ubuntu 18.04에서는 지원함.
- tomcat 데몬관리자인 `jsvc`로 실행하는 방법이 있었으나 패키지 관리자에 의해 설치된 환경을 최대한 이용함.
- libtcnative-1 을 recommands로 함께 설치함
> APR(Apache Portable Runtime)을 직접 사용하지는 않지만 TLS 프로토콜 운영시 OpenSSL 라이브러리와 연동 가능 
> 성능 향상 및 관리 측면 유연성 증가 
> tomcat 설치시 `postinstall` 스크립트에 의해 tomcat8 데몬이 구동 되어지는데 
> 이 상태에서 CATALINA_BASE 를 추가 구성해 재기동을 하면 기존 데몬을 탐지 하지 못하는 이슈가 있다. 
> 때문에 [policy-rc.d](https://jpetazzo.github.io/2013/10/06/policy-rc-d-do-not-start-services-automatically/)를 설정하여 설치후 데몬이 구동되지 않도록 한다. 

### configuration
- 멀티 instance운영을 지양하나 유연성을 갖추기 위해 instance단위로 설정
- CATALINA_HOME을 이용하지 않고 각 어플리케이션 단위로 CATALINA_BASE를 갖을수 있도록 함.


#### user
- 전용 계정을 생성 하여 운영 : `tomcat`
```
  tomcat:
    user:
      name: tomcat
      id: 990
    group:
      name: tomcat
      id: 990

```
- login shell 및 별도 homedir을 갖지 않음
- 추후 NFS 공유시 발생할수 있는 UID/GID 미스매치를 완화 함
- 운영중 변경은 권장하지 않는다.
  - 필요시 수동으로 트래픽 유입을 차단후 수동 작업한다.
  ```bash
      tomcat_old_uid=$(id -u tomcat)
      tomcat_old_gid=$(id -g tomcat)
      killall -u tomcat
      usermod -u 998 tomcat
      groupmode -u 998 tomcat
      find / -path /sys -prune -o -path /proc -prune -o -group $tomcat_old_gid -exec chgrp -h tomcat {} \;
      find / -path /sys -prune -o -path /proc -prune -o -user  $tomcat_old_uid -exec chown -h tomcat {} \;
  ```


#### java
```
sudo apt update
sudo apt install openjdk-8-jre-headless
```
- 버전 명세
  - JAVA_HOME 변수를 지정하여 사용
  - /var/lib/tomcat9/bin/setenv.d/10-javahome.sh
  ```bash
  JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
  ```
> `update-alternatives`, `update-java-alternatives`등으로 시스템 전역 설정을 이용하지 않음
> `10-javahome.sh`를 전용으로 사용하여 해당 애플리케이션에만 지역적으로 자바버전을 설정함
(자바 버전 전역 설정으로 인한 사이드이펙 방지)


#### bin/setenv.sh
```bash
for add_env in $(ls $CATALINA_BASE/bin/setenv.d/*.sh)
do
    test -r $add_env && . $add_env
done
```
- setenv.sh 에서 직접 설정하지 않는다.
- setenv.d/*.sh 를 순회하여 설정한다.
- 필요한 항목별로 분리 해서 설정하며
- \d\d-\w\w\w\w.conf 의 형식을 사용한다.
> oraclejdk 에서는 java 옵션을 파싱할때 right most로 파싱하여 중복 되는 파라미터를 처리 하는데
> openjdk에서는 중복 선언 오류를 발생 시킨다.

#### conf/server.xml
- APR 리스너를 추가 하지만 프로토콜은 비활성화 한다.(apache 사용하지 않음)
- 공통 executor를 만들어서 각 connector에서 상속 받는다.
- Connector는 nio1 를 사용한다.
- HTTP2 버전을 활성화 한다.
- SSL 설정을 가능하게 하며, OpenSSL을 사용한다.
  - [ ] JSSE implementation provided as part of the Java runtime
  - [x] JSSE implementation that uses OpenSSL
  - [ ] APR implementation, which uses the OpenSSL engine by default

#### logging & rotation
패키지와 함께 제공되는 로그로테이션 설정은 CATALINA_BASE에 구성된 인스턴스 로그에는 적용되어지지 않아 추가 설정이 필요하여 설정후 `/etc/logrotate.d/tomcat-{{ service }}`로 추가함.
```
## made by ansible
/opt/{{ service }}/logs/catalina.out
/opt/{{ service }}/logs/access.log {
    rotate 1000
    maxage 7
    size 100M
    copytruncate
    olddir /opt/{{ service }}/logs/old
    compress
    missingok
    create 640
    notifempty
}
```
accesslog의 로그형식
```xml
<Valve className="org.apache.catalina.valves.AccessLogValve"
	rotatable="false"
	buffered="false"
	directory="logs"
	prefix="access"
	suffix=".log"
	pattern='%a %l %u "%{yyyy-MM-dd}tT%{HH:mm:ssZ}t" "%r" %s %b "%{Referer}i" "%{User-Agent}i" "%{Cookie}i" "%{X-Forwarded-For}i" %I %v %D'
	resolveHosts="false" />
```

#### jmxremote
[visualvm](https://visualvm.github.io/) 등을 이용하여 구동중인 jvm 환경에 디버깅, 모니터링 할 수 있도록 jvmremote 활성화.
- 계정 : sys-jmx
- 패스워드 : sys-jmx
- port : 38080
- 권한 : read only
- 주소 : 서버주소

## how to run tomcat-playbook with sample.war for testing
```bash
$ git clone https://somewhere.git
$ INVEN_HOST='10.100.0.70'
$ INVEN_PORT=':65422'
$ ansible-playbook -i $INVEN_HOST$INVEN_PORT, ./tomcat85/tomcat85.yml -e "service=sample ansible_python_interpreter=/usr/bin/python3" -e "deploy=yes"
$ curl -vvv http://$INVEN_HOST/
$ curl -vvv https:/sample.site/ --resolve sample.site:443:$INVEN_HOST
```
