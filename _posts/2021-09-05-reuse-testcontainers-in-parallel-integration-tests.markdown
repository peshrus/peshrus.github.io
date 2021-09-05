---
layout: post
title:  "Reuse test containers in parallel integration tests"
date:   2021-09-05 09:13:00 +0200
categories: testcontainers integration-tests
---

## Motivation

Use the same [testcontainers](https://www.testcontainers.org/) in all integration tests running in
parallel processes. By default, each process starts its own containers.

## Prerequisites

- The integration tests are configured as described in
the [Gradle documentation](https://docs.gradle.org/current/userguide/java_testing.html#sec:configuring_java_integration_tests).
- The code is prepared to use the same containers:
  - Each test is started in its own DB transaction to make the changes invisible to other tests and
    to roll the data back at the end
  - Each test has its own Mongo DB (if Mongo is used) that is deleted at the end of the test
  - Each test has its own Elasticsearch index (if ES is used) that is deleted at the end of the test

## Implementation

### Gradle

```groovy
def runItDocker = tasks.register("runItDocker", JavaExec) {
    it.outputs.cacheIf { true }
    it.inputs.files(
            fileTree("${project.buildDir}/classes/kotlin/main"),
            fileTree("${project.buildDir}/classes/kotlin/integrationTest"),
            fileTree("${project.buildDir}/resources/main"),
            fileTree("${project.buildDir}/resources/integrationTest")
    )
            .withPropertyName("sourceFiles")
            .withPathSensitivity(PathSensitivity.RELATIVE)
    it.outputs.files(file("${System.getProperty("user.home")}/.testcontainers.properties"))
            .withPropertyName("outputFiles")
    it.group = "<YOUR_GROUP>"
    it.description = "Run integration test docker containers."
    it.mainClass.set("ContainersKt")
    it.classpath = sourceSets.integrationTest.runtimeClasspath
    // See https://www.testcontainers.org/features/configuration/#disabling-ryuk
    it.environment("TESTCONTAINERS_RYUK_DISABLED", "true")
    it.doFirst {
        setTestContainersProperties()
    }
}

// Reusable containers stay running after the integration tests finish.
// This task is intended to stop these Docker containers manually.
def stopItDocker = tasks.register("stopItDocker", Exec) {
    it.group = "<YOUR_GROUP>"
    it.description = "Stop integration test docker containers."
    it.doFirst {
        it.commandLine "./stopItDocker.sh"
    }
}

def integrationTest = tasks.register("integrationTest", Test) {
    it.dependsOn runItDocker
    it.group = "verification"
    it.description = "Integration tests."

    it.forkEvery = 250
    it.maxParallelForks = 2

    it.testClassesDirs = sourceSets.integrationTest.output.classesDirs
    it.classpath = sourceSets.integrationTest.runtimeClasspath
    // See https://www.testcontainers.org/features/configuration/#disabling-ryuk
    it.environment("TESTCONTAINERS_RYUK_DISABLED", "true")
    it.shouldRunAfter test

    it.finalizedBy stopItDocker
}

@groovy.transform.CompileStatic
void setTestContainersProperties() {
    final def properties = new Properties()
    final def propertiesFile = file("${System.getProperty("user.home")}/.testcontainers.properties")

    if (propertiesFile.exists()) {
        properties.load(propertiesFile.newDataInputStream())
    }

    if (properties.getProperty("testcontainers.reuse.enable") == "true" &&
            properties.getProperty("checks.disable") == "true") {
        return
    }

    properties.setProperty("testcontainers.reuse.enable", "true")
    // See https://www.testcontainers.org/features/configuration/#disabling-the-startup-checks
    properties.setProperty("checks.disable", "true")

    propertiesFile.withOutputStream { stream ->
        properties.store(stream, null)
    }
}
```

### Kotlin

```kotlin
import DbSpec.Companion.rdsEnvironmentMap
import io.kotlintest.extensions.system.withEnvironment
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.joinAll
import kotlinx.coroutines.launch
import org.testcontainers.containers.BindMode
import org.testcontainers.containers.GenericContainer
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.containers.localstack.LocalStackContainer
import org.testcontainers.containers.wait.strategy.Wait
import org.testcontainers.elasticsearch.ElasticsearchContainer
import org.testcontainers.utility.DockerImageName
import utils.KGenericContainer
import java.nio.file.Paths
import java.util.Collections.singletonMap

// See:
// - https://medium.com/vattenfall-tech/optimise-testcontainers-for-better-tests-performance-20a131d6003c
// - https://rieckpil.de/reuse-containers-with-testcontainers-for-fast-integration-tests/
object Containers {
    private const val IMG_POSTGRES = "postgres:12.6-alpine"
    private const val IMG_MONGO = "mongo:4.2"
    private const val IMG_ELASTICSEARCH = "docker.elastic.co/elasticsearch/elasticsearch:7.10.0"

    val postgresInstance by lazy { startPostgresContainer() }
    val mongoInstance by lazy { startMongoDbContainer() }
    val esInstance by lazy { startEsContainer() }

    private fun makeTmpFs(folderName: String) = singletonMap(makeTmpPath(folderName), "rw")

    private fun makeTmpPath(folderName: String) = Paths.get(System.getProperty("java.io.tmpdir"), folderName).toString()

    // Use PostgreSQL container to allow us to set credentials and initialise DB
    class MyPostgreSQLContainer(imageName: String) : PostgreSQLContainer<MyPostgreSQLContainer>(imageName)

    // Change the version together with docker-compose.yml:4
    private fun startPostgresContainer() = MyPostgreSQLContainer(IMG_POSTGRES).apply {
        withDatabaseName("<DB_NAME>")
        withUsername("postgres")
        withPassword("password")
        setWaitStrategy(Wait.forLogMessage(".*init process complete.*", 1))
        startOneInstance(
            name = "<PREFIX>-it-postgres",
            tmpFolderName = "postgresql_data",
            command = arrayOf("postgres", "-c", "max_connections=500")
        )
    }

    private fun startMongoDbContainer() = KGenericContainer(IMG_MONGO).apply {
        withExposedPorts(27017)
        startOneInstance(name = "<PREFIX>-it-mongo", tmpFolderName = "mongodb_data")
    }

    private fun startEsContainer() = ElasticsearchContainer(IMG_ELASTICSEARCH).apply {
        addExposedPorts(9200, 9300, 33072)
        addEnv("cluster.name", "elasticsearch")
        addEnv("http.compression", "true")
        addEnv("http.compression_level", "1")
        addEnv("transport.compress", "true")
        addEnv("logger.org.elasticsearch", "ERROR")
        startOneInstance(name = "<PREFIX>-it-elasticsearch", tmpFolderName = "es_data")
    }

    // See https://docs.gradle.org/current/dsl/org.gradle.api.tasks.testing.Test.html#org.gradle.api.tasks.testing.Test:maxParallelForks
    // The tests are started in parallel processes.
    // To prevent the creation of multiple containers for the same image every container has a fixed name.
    // This implementation allows reusing the same container between tests in parallel processes.
    private fun GenericContainer<*>.startOneInstance(name: String, tmpFolderName: String, vararg command: String) {
        withReuse(true)
        withCreateContainerCmdModifier { cmd ->
            cmd.withName(name)
            if (command.isNotEmpty()) {
                cmd.withCmd(*command)
            }
        }
        // Container can have tmpfs mounts for storing data in host memory.
        // This is useful if you want to speed up your database tests.
        withTmpFs(makeTmpFs(tmpFolderName))
        start()
    }
}

/**
 * Used in build.gradle.
 * Starts all containers before the integration tests and runs Flyway migrations.
 */
suspend fun main(): Unit = coroutineScope {
    listOf(
        launch(Dispatchers.IO) {
            Containers.postgresInstance
            // Run DB migrations
        },
        launch(Dispatchers.IO) {
            Containers.esInstance
        },
        launch(Dispatchers.IO) {
            Containers.mongoInstance
        }
    ).joinAll()
}
```

### Bash

**stopItDocker.sh**:
```shell
#!/bin/bash
# Remove all integration test docker containers with volumes
#
# -a - all containers
# -f "name=<PREFIX>-it-" - having the name
# -q - quiet
containers=$(docker ps -af "name=<PREFIX>-it-" -q)
if [ ! -z "$containers" ]; then # if not empty
  # -f - force
  # -v - with volumes
  echo $containers | xargs docker rm -fv > /dev/null 2>&1 &
fi
```

## Conclusion

This approach allows not only to run the integration tests reusing the same test containers in
parallel processes but also guarantees the independence of the tests from each other, they are run
with isolated data sets (in case the tests are prepared accordingly, see Prerequisites).  
