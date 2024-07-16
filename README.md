# Prepare a Spring Boot app with CRaC
For our tutorial, we will use a Spring Boot reference application, Spring Petclinic.

First of all, clone the Spring Petclinic's repository:
```
git clone https://github.com/spring-projects/spring-petclinic.git
```
To use CRaC with the new Spring Boot and Spring Framework, we need to add the dependency for the org.crac/crac package to pom.xml:

```
<dependency>
    <groupId>org.crac</groupId>
    <artifactId>crac</artifactId>
    <version>1.4.0</version>
</dependency>
```
Let's check the difference of our updated pom.xml:
```
git diff
diff --git a/pom.xml b/pom.xml
index 287a08a..f403155 100644
--- a/pom.xml
+++ b/pom.xml
@@ -36,6 +36,11 @@
   </properties>

   <dependencies>
+    <dependency>
+        <groupId>org.crac</groupId>
+        <artifactId>crac</artifactId>
+        <version>1.4.0</version>
+    </dependency>
     <!-- Spring and Spring Boot dependencies -->
     <dependency>
       <groupId>org.springframework.boot</groupId>
```
Now, build the application with

```
mvn clean package
```
# Prepare a work directory for the application dump

CRaC enables the developers to save the exact state of a running Java application (together with the information about the heap, JIT-compiled code, and so on). We need to create a work directory where the application dump will be stored after the checkpoint.

For this purpose, run

```
mkdir -p storage/checkpoint-spring-petclinic
```
We should also copy the Petclinic's jar file to the “storage” to use in a docker container:


```
cp target/spring-petclinic-3.2.0-SNAPSHOT.jar ./storage/
```
# Start the application in a Docker container

We will use the bellsoft/liberica-runtime-container:jdk-21-crac-slim-glibc image to start Petclinic and get the application dump for further restore. Note that BellSoft also provides images with musl libc.

Another important prerequsite is the kernel version of the underlying Linux distribution, which should be at least 5.9. Linux kernel 5.9 introduces the CAP_CHECKPOINT_RESTORE option, which separates the checkpoint/restore functionality from the CAP_SYS_ADMIN option and enables the developers to steer clear of running containers with elevated permissions.

The CAP_CHECKPOINT_RESTORE (and SYS_PTRACE, which is also required for checkpoint-restore) options are enabled with --cap-add flag added to newer Docker versions, so make sure you have the latest available Docker version.

Use the following command to start the application in a Docker container:

```
docker run --cap-add CHECKPOINT_RESTORE --cap-add SYS_PTRACE -d -v $(pwd)/storage:/storage -w /storage --name petclinic-app-container bellsoft/liberica-runtime-container:jdk-21-crac-slim-glibc java -Xmx512m -XX:CRaCCheckpointTo=/storage/checkpoint-spring-petclinic -jar spring-petclinic-3.3.0-SNAPSHOT.jar
```
If you use a Linux distribution with an older kernel version, you can you the --priviledged flag instead of CAP_CHECKPOINT_RESTORE and SYS_PTRACE (however, this is not the best practice to run containers with elevated permissions).

Now, let’s check the application output.

```
docker logs petclinic-app-container
```
It is also possible to warm up the HotSpot Virtual Machine to get the compiled hot code for better performance. This can be done on the host machine by using HTTP requests to Petclinic endpoints. The port should be exposed to the host machine using -p 8080:8080 options, for example. 

Having started the container, let’s check the application resident set memory size (RSS). To do that, get the application PID at first:


```
docker exec petclinic-app-container ps -a | grep spring-petclinic | tail -1
```
The output will contain the application PID:


```
129 root 0:23 java -Xmx512m -XX:CRaCCheckpointTo=/storage/checkpoint-spring-petclinic -jar spring-petclinic-3.2.0-SNAPSHOT.jar
```
Note that the PID can also be obtained from the application log or by using the jcmd command.

Now, we can check the resident set memory size (RSS):


```
docker exec petclinic-app-container  pmap -x 129 | tail -1
total            5447120  373618  352624       0
```
The second value (373618) is RSS in Kb.

In addition, you can check the /proc filesystem:

```
docker exec petclinic-app-container cat /proc/129/statm
1361779 93498 5440 1 0 119242 0
```
The second value is RSS in pages. The page size is usually 4k, and it can be checked with the command docker exec petclinic-app-container getconf PAGESIZE, with the possible output of 4096.

# Perform the checkpoint to get the application dump
The jcmd command is used to send the command to VM to make the checkpoint and dump the application and VM state to storage:

```
docker exec petclinic-app-container jcmd spring JDK.checkpoint
```
As a result, the Java instance with the Petclinic app will be dumped and the container will be stopped. The docker ps will show that the petclinic-app-container was stopped.

You can check that the images for your application were dumped to the directory:

```
ls storage/checkpoint-spring-petclinic/

core-129.img  core-133.img  core-137.img  core-141.img  core-145.img  core-151.img  core-156.img  core-160.img  core-165.img  core-208.img  core-212.img  fs-129.img     pagemap-129.img  stats-dump
core-130.img  core-134.img  core-138.img  core-142.img  core-146.img  core-152.img  core-157.img  core-161.img  core-166.img  core-209.img  dump4.log     ids-129.img    pages-1.img      timens-0.img
core-131.img  core-135.img  core-139.img  core-143.img  core-147.img  core-153.img  core-158.img  core-163.img  core-167.img  core-210.img  fdinfo-2.img  inventory.img  pstree.img
core-132.img  core-136.img  core-140.img  core-144.img  core-150.img  core-154.img  core-159.img  core-164.img  core-207.img  core-211.img  files.img     mm-129.img     seccomp.img
```

# Use the prepared dump to start the application and the VM

Let’s start the application by restoring it from the dump. To do that, run


```
docker run --cap-add CHECKPOINT_RESTORE --cap-add SYS_PTRACE -it --rm -v $(pwd)/storage:/storage -w /storage -p 8080:8080 --name petclinic-app-container-from-checkpoint bellsoft/liberica-runtime-container:jdk-21-crac-slim-glibc java -XX:CRaCRestoreFrom=/storage/checkpoint-spring-petclinic
```
Please note that the -p 8080:8080 option is added to expose the port for the running application.

Check the application logs:


```
$ docker logs petclinic-app-container-from-checkpoint
2024-03-01T17:34:58.675Z  WARN 129 --- [l-1 housekeeper] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Thread starvation or clock leap detected (housekeeper delta=6m21s375ms900µs465ns).
2024-03-01T17:34:58.678Z  INFO 129 --- [Attach Listener] o.s.c.support.DefaultLifecycleProcessor  : Restarting Spring-managed lifecycle beans after JVM restore
2024-03-01T17:34:58.684Z  INFO 129 --- [Attach Listener] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
2024-03-01T17:34:58.685Z  INFO 129 --- [Attach Listener] o.s.c.support.DefaultLifecycleProcessor  : Spring-managed lifecycle restart completed (restored JVM running for 27 ms)
```
Let's check the application RSS before the first request.


```
docker exec petclinic-app-container-from-checkpoint pmap -x 129 | tail -1
total            5646672  212488   70768       0
```
It is also possible to start other Petclinic instances in separate containers — just use another port on the host system and another container name. For example:


```
docker run --cap-add CHECKPOINT_RESTORE --cap-add SYS_PTRACE -it --rm -d -v $(pwd)/storage:/storage -w /storage -p 8081:8080 --name petclinic-app-container-from-checkpoint2 bellsoft/liberica-runtime-container:jdk-21-crac-slim-glibc java -XX:CRaCRestoreFrom=/storage/checkpoint-spring-petclinic
```
To verify that the app is working correctly, open http://localhost:8080. If there are other instances started, you can open Spring Petclinic using a different port, for example, http://localhost:8081 if the port is opened as in the previous command.

Let’s check the application RSS after the first request:
```
docker exec petclinic-app-container-from-checkpoint pmap -x 129 | tail -1
total            5646412  252653  198904       0
```
We can see that RSS before the dump is larger than RSS after the restore. After the first request, RSS increases. However, it is still less than the original RSS before dumping. It happens for two reasons:

Liberica JDK with CRaC performs full Garbage Collection and returns a part of native memory to the OS.
While criu (the executable that is used by Liberica JDK with CRaC to perform dumping) is working, some pages that belong to Java Virtual Machine or used by system libraries are not used after the restore. The memory pages can still be used by libraries or JVM, but after restoring thay are just reserved for further use.
As you can see, CRaC API can be conveniently used with your containerized workloads thanks to ready-to-use Alpaquita Containers.
