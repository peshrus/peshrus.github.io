---
layout: post
title:  "How to check Liquibase scripts using Docker during Maven build"
date:   2020-12-29 15:21:00 +0100
categories: liquibase docker maven
---

The example below shows how to run the project Liquibase changelog files against Oracle DB. The
same approach can be used with any database. Enjoy!

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>io.github.peshrus</groupId>
  <artifactId>liquibase-docker-maven</artifactId>
  <version>1.0-SNAPSHOT</version>

  <build>
    <plugins>
      <!-- Copy JARs needed for Liquibase to the target folder to use them in the classpath -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-dependency-plugin</artifactId>
        <version>3.1.2</version>
        <executions>
          <execution>
            <id>copy</id>
            <phase>package</phase>
            <goals>
              <goal>copy</goal>
            </goals>
            <configuration>
              <artifactItems>
                <artifactItem>
                  <groupId>com.oracle.database.jdbc</groupId>
                  <artifactId>ojdbc10</artifactId>
                  <version>19.9.0.0</version>
                  <overWrite>true</overWrite>
                  <outputDirectory>${project.build.directory}/lib</outputDirectory>
                  <destFileName>ojdbc10.jar</destFileName>
                </artifactItem>
                <!-- You may need other artifactItems. In my case, I also needed trove4j and liquibase-core -->
              </artifactItems>
            </configuration>
          </execution>
        </executions>
      </plugin>

      <!-- Runs integration tests. docker-maven-plugin is attached by default to integration test lifecycle phases (see https://github.com/fabric8io/docker-maven-plugin) -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-failsafe-plugin</artifactId>
        <version>3.0.0-M4</version>
        <configuration>
          <skipITs>false</skipITs>
        </configuration>
        <executions>
          <execution>
            <goals>
              <goal>integration-test</goal>
              <goal>verify</goal>
            </goals>
          </execution>
        </executions>
      </plugin>

      <!-- Generates reports for the integration tests -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-report-plugin</artifactId>
        <version>3.0.0-M4</version>
        <executions>
          <execution>
            <phase>post-integration-test</phase>
            <goals>
              <goal>failsafe-report-only</goal>
            </goals>
          </execution>
        </executions>
      </plugin>

      <!-- Runs Oracle and Liquibase Docker images -->
      <plugin>
        <groupId>io.fabric8</groupId>
        <artifactId>docker-maven-plugin</artifactId>
        <version>0.34.1</version>
        <configuration>
          <images>
            <image>
              <alias>oracle</alias>
              <name><!-- YOUR ORACLE IMAGE --></name>
              <run>
                <!-- This hostname is used below in JDBC URL configuration -->
                <hostname>oracle_db</hostname>
                <wait>
                  <log>Port 1521</log>
                  <time>20000</time>
                </wait>
                <log>
                  <date>default</date>
                  <color>magenta</color>
                </log>
              </run>
            </image>
            <image>
              <alias>liquibase</alias>
              <name>liquibase/liquibase</name>
              <run>
                <dependsOn>
                  <container>oracle</container>
                </dependsOn>
                <links>
                  <link>oracle:db</link>
                </links>
                <env>
                  <_JAVA_OPTIONS>-Doracle.jdbc.timezoneAsRegion=false</_JAVA_OPTIONS>
                </env>
                <volumes>
                  <bind>
                    <!-- Mapping of the folder where we have copied the libraries to the Liquibase container path -->
                    <volume>${project.build.directory}/lib:/liquibase/lib</volume>
                    <!-- Mapping of the folder with Liquibase changelog files to the Liquibase container path -->
                    <volume>${project.basedir}:/liquibase/changelog</volume>
                  </bind>
                </volumes>
                <cmd>--classpath=lib/ojdbc10.jar:changelog/target/classes:changelog
                  --driver=oracle.jdbc.driver.OracleDriver
                  --url=jdbc:oracle:thin:@oracle_db:1521:YOUR_SCHEMA
                  --changeLogFile=changelog/target/classes/PATH_TO_YOUR_DB_CHANGE_LOG.xml
                  --username=YOUR_DB_USER_NAME
                  --password=YOUR_DB_USER_PASSWORD
                  --logLevel=severe
                  update
                </cmd>
                <wait>
                  <log>Liquibase: Update has been successful.</log>
                  <time>30000</time>
                </wait>
                <log>
                  <date>default</date>
                  <color>cyan</color>
                </log>
              </run>
            </image>
          </images>
        </configuration>
        <executions>
          <execution>
            <id>start</id>
            <phase>pre-integration-test</phase>
            <goals>
              <goal>start</goal>
            </goals>
          </execution>
          <execution>
            <id>stop</id>
            <phase>post-integration-test</phase>
            <goals>
              <goal>stop</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>

  <!-- Generates reports for the integration tests -->
  <reporting>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-report-plugin</artifactId>
        <version>3.0.0-M4</version>
        <reportSets>
          <reportSet>
            <id>integration-tests</id>
            <reports>
              <report>failsafe-report-only</report>
            </reports>
          </reportSet>
        </reportSets>
      </plugin>
    </plugins>
  </reporting>
</project>
```
