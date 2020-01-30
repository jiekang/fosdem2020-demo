# fosdem2020-demo

This project contains instructions for deploying on a CRC instance (Code Ready Containers, OpenShift 4.x local deployment) the following:
* problematic-microservices (https://github.com/jiekang/problematic-microservices : forked from Marcus Hirt's project, with a branch to easily create images)
* container-jfr via Operator
* Jaeger via Operator

This requires access to an OpenShift cluster with both cli tool `oc` as well as the web UI.

Create a project for the services to be deployed with:
```
oc new-project fosdem2020
```

## problematic-microservices

The problematic-microservices consists of three services:
* customer-service
* factory-service 
* order-service

To deploy these from quay.io images, run:

```
// customer-service
oc new-app quay.io/jiekang/robotshop-customer-service --name=customer-service-1 \
-e PORT=8081 \
-e JAEGER_ENDPOINT=http://jaeger-all-in-one-inmemory-collector:14268/api/traces \
-e JAVA_OPTS="-Dcom.sun.management.jmxremote.port=9091 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -XX:+FlightRecorder -XX:FlightRecorderOptions=stackdepth=128 -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints -Djava.net.preferIPv4Stack=true -XX:StartFlightRecording=name=CustomerService,path-to-gc-roots=true,settings=profile,maxsize=128MB"
oc expose svc/customer-service-1

// factory-service
oc new-app quay.io/jiekang/robotshop-factory-service --name=factory-service-1 \
-e PORT=8081 \
-e JAEGER_ENDPOINT=http://jaeger-all-in-one-inmemory-collector:14268/api/traces \
-e "JAVA_OPTS=-Dcom.sun.management.jmxremote.port=9091 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -XX:+FlightRecorder -XX:FlightRecorderOptions=stackdepth=128 -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints -Djava.net.preferIPv4Stack=true -XX:StartFlightRecording=name=FactoryService,path-to-gc-roots=true,settings=profile,maxsize=128MB"
oc expose svc/factory-service-1

// order-service
oc new-app quay.io/jiekang/robotshop-order-service --name=order-service-1 \
-e PORT=8081 \
-e JAEGER_ENDPOINT=http://jaeger-all-in-one-inmemory-collector:14268/api/traces \
-e CUSTOMER_SERVICE_LOCATION=http://customer-service-1-fosdem2020.apps-crc.testing \
-e FACTORY_SERVICE_LOCATION=http://factory-service-1-fosdem2020.apps-crc.testing \
-e "JAVA_OPTS=-Dcom.sun.management.jmxremote.port=9091 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -XX:+FlightRecorder -XX:FlightRecorderOptions=stackdepth=128 -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints -Djava.net.preferIPv4Stack=true -XX:StartFlightRecording=name=OrderService,path-to-gc-roots=true,settings=profile,maxsize=128MB"
oc expose svc/order-service-1
```

Once the services are made available, you can use the load generator to target them via, for example:
```
CUSTOMER_SERVICE_LOCATION=http://customer-service-1-fosdem2020.apps-crc.testing FACTORY_SERVICE_LOCATION=http://factory-service-1-fosdem2020.apps-crc.testing ORDER_SERVICE_LOCATION=http://order-service-1-fosdem2020.apps-crc.testing java11 -jar target/app.jar
```

## container-jfr via Operator

The container-jfr-operator is currently available as a custom Operator, from the upstream repository. Add it to the cluster by:

```
git clone https://github.com/rh-jmc-team/container-jfr-operator.git
oc apply container-jfr-operator/bundle/openshift/operator-source.yaml
```

Afterwards you can install the Operator through the cluster's OperatorHub, targeting the OpenShift project containing the Java applications.

Once the Operator is installed, create a ContainerJFR instance with `minimal:false`, e.g.

```
apiVersion: rhjmc.redhat.com/v1alpha1
kind: ContainerJFR
metadata:
  name: example
  namespace: fosdem2020
spec: {
  minimal: false
}
```

## Jaeger via Operator

Jaeger is available on the OperatorHub. Install the Operator (Red Hat version rather than community edition), targeting the OpenShift project containing the Java applications.

Once the Operator is installed, create a Jaeger instance with the default jaeger-all-in-one-inmemory definition.