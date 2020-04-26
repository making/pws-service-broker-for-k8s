# Deploy PWS Marketplace on cf-for-k8s


## Deploy the Service Broker

```
cp sample-values.yml values.yml
# fill values.yml
kapp -a pws-service-broker deploy -c -f <(ytt -f config -f values.yml)
```

## Register the Service Broker

```
cf create-service-broker pws broker broker http://pws-service-broker.pws-service-broker.svc.cluster.local:8080
```

Check available services and plans.

```
$ cf service-access
Getting service access as admin...
broker: pws
   service              plan                             access   orgs
   app-autoscaler       standard                         none     
   cedexisopenmix       openmix-gslb-with-fusion-feeds   none     
   cedexisopenmix       opx_global                       none     
   cleardb              amp                              none     
   cleardb              boost                            none     
   cleardb              shock                            none     
   cleardb              spark                            none     
   cloudamqp            bunny                            none     
   cloudamqp            lemur                            none     
   cloudamqp            panda                            none     
   cloudamqp            rabbit                           none     
   cloudamqp            tiger                            none     
   elephantsql          elephant                         none     
   elephantsql          hippo                            none     
   elephantsql          panda                            none     
   elephantsql          turtle                           none     
   gluon                business                         none     
   gluon                enterprise                       none     
   gluon                free                             none     
   gluon                indie                            none     
   Greenplum            Free                             none     
   loadimpact           li100                            none     
   loadimpact           li1000                           none     
   loadimpact           li500                            none     
   loadimpact           lifree                           none     
   memcachedcloud       100mb                            none     
   memcachedcloud       1gb                              none     
   memcachedcloud       2-5gb                            none     
   memcachedcloud       250mb                            none     
   memcachedcloud       30mb                             none     
   memcachedcloud       500mb                            none     
   memcachedcloud       5gb                              none     
   memcachier           100                              none     
   memcachier           10000                            none     
   memcachier           1000                             none     
   memcachier           100000                           none     
   memcachier           20000                            none     
   memcachier           2000                             none     
   memcachier           250                              none     
   memcachier           5000                             none     
   memcachier           500                              none     
   memcachier           50000                            none     
   memcachier           7500                             none     
   memcachier           dev                              none     
   mlab                 sandbox                          none     
   newrelic             lite                             none     
   p-cloudcache         dev-plan                         none     
   p-cloudcache         extra-small                      none     
   p-cloudcache         small                            none     
   p-cloudcache         small-footprint                  none     
   p-config-server      standard                         none     
   p-config-server      trial                            none     
   p-identity           springdev                        none     
   p-identity           wtran-springone                  none     
   p-service-registry   standard                         none     
   p-service-registry   trial                            none     
   p.mysql              db-large                         none     
   p.mysql              db-lf-small                      none     
   p.mysql              db-medium                        none     
   p.mysql              db-small                         none     
   p.redis              cache-large                      none     
   p.redis              cache-medium                     none     
   p.redis              cache-small                      none     
   pubnub               free                             none     
   quotaguard           deluxe                           none     
   quotaguard           enterprise                       none     
   quotaguard           large                            none     
   quotaguard           medium                           none     
   quotaguard           mega                             none     
   quotaguard           micro                            none     
   quotaguard           premium                          none     
   quotaguard           spike                            none     
   quotaguard           starter                          none     
   quotaguard           super                            none     
   quotaguard           unlimited                        none     
   rediscloud           100mb                            none     
   rediscloud           10gb                             none     
   rediscloud           1gb                              none     
   rediscloud           2-5gb                            none     
   rediscloud           250mb                            none     
   rediscloud           30mb                             none     
   rediscloud           500mb                            none     
   rediscloud           50gb                             none     
   rediscloud           5gb                              none     
   scheduler-for-pcf    standard                         none     
   searchify            plus                             none     
   searchify            pro                              none     
   searchify            small                            none     
   searchly             advanced                         none     
   searchly             business                         none     
   searchly             enterprise                       none     
   searchly             micro                            none     
   searchly             professional                     none     
   searchly             small                            none     
   searchly             starter                          none     
   sendgrid             bronze                           none     
   sendgrid             free                             none     
   sendgrid             silver                           none     
   ssl                  basic                            none     
   streamdata           brook                            none     
   streamdata           creek                            none     
   streamdata           spring                           none
```

Enable `cleardb` and `elephantsql`.

```
cf enable-service-access cleardb
cf enable-service-access elephantsql
```

Check marketplace to see `cleardb` and `elephantsql` are available .

```
$ cf marketplace                      
Getting services from marketplace in org demo / space demo as admin...
OK

service       plans                            description                             broker
cleardb       spark, boost, amp, shock         Highly available MySQL for your Apps.   pws
elephantsql   turtle, panda, hippo, elephant   PostgreSQL as a Service                 pws

TIP: Use 'cf marketplace -s SERVICE' to view descriptions of individual plans of a given service.
```

## Deploy a sample application

```
curl https://start.spring.io/starter.tgz \
       -d artifactId=hello-db \
       -d baseDir=hello-db \
       -d dependencies=data-rest,data-jpa,data-rest-hal,actuator,mysql \
       -d packageName=com.example \
       -d applicationName=HelloDbApplication | tar -xzvf -

cd hello-db

cat <<EOF > src/main/resources/application.properties
spring.jpa.hibernate.ddl-auto=update
spring.datasource.url=jdbc:mysql://localhost:3306/demo
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
EOF
cat <<EOF > src/main/java/com/example/Message.java
package com.example;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
public class Message {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    public Long id;
    public String text;
}
EOF
cat <<EOF > src/main/java/com/example/MessageRepository.java
package com.example;

import org.springframework.data.repository.CrudRepository;

public interface MessageRepository extends CrudRepository<Message, Long> {
}
EOF
```

Build the app.

```
./mvnw clean package -DskipTests
```

Create a service instance of `cleardb`.

```
cf create-service cleardb spark message-db
```

Check service instances.

```
$ cf services
Getting services in org demo / space demo as admin...

name         service   plan    bound apps   last operation     broker   upgrade available
message-db   cleardb   spark   hello-db     create succeeded   pws  
```

Create `maniefst.yml`.

```
cat <<'EOF' > manifest.yml
applications:
- name: hello-db
  path: target/hello-db-0.0.1-SNAPSHOT.jar
  services:
  - message-db
  env:
    BP_AUTO_RECONFIGURATION: false
    SPRING_DATASOURCE_URL: jdbc:mysql://${vcap.services.message-db.credentials.hostname}:${vcap.services.message-db.credentials.port}/${vcap.services.message-db.credentials.database}
    SPRING_DATASOURCE_USERNAME: ${vcap.services.message-db.credentials.username}
    SPRING_DATASOURCE_PASSWORD: ${vcap.services.message-db.credentials.password}
EOF
```

Deploy the app.

```
cf push
```

Access endpoints.

```
curl  -H 'Content-Type: application/json' -d '{"text":"Hello World!"}'  hello-db.local.maki.lol/messages
curl hello-db.local.maki.lol/messages
```