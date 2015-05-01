Using JaCoCo with the Solano Data Warehouse
===========================================
The spring-mvc-showcase example has been instrumented with JaCoCo so that code coverage is performed during testing.

The project contains both a test and integration-test folder.  The test folder contains all of the unit tests that came with the source project.  The integration-test folder was created to show use of the JaCoCo merge feature that will merge a set of .exec coverage files into a single merged coverage file as well as a single set of rendered coverage views (including html, css, xml and csv files).  The integration-test folder contains a single test named ITConvertControllerTests.  It was previously named ConvertControllerTests and lived under the test folder.  It was moved here to force a second test pass to run that would generate the it.exec file that is mergerd with the ut.exec file to produce the merged.exec file containing the rolled up code coverage results for this project.

Running the build to run all tests and package up the results is done with the command line:  mvn clean verify test -P all-tests

The all-tests profile insures both the "test" and "integration-test" targets are run.  The JaCoCo merge goal is run once the tests have completed running.

JaCoCo merge configuration:

          <execution>
            <id>merge</id>
            <phase>post-integration-test</phase>
            <goals>
              <goal>merge</goal>
            </goals>
            <configuration>
              <destFile>${jacoco.merge.execution.data.file}</destFile>
              <fileSets>
                <fileSet implementation="org.apache.maven.shared.model.fileset.FileSet">
                  <directory>${project.build.directory}/coverage-reports</directory>
                  <includes>
                    <include>*.exec</include>
                  </includes>
                </fileSet>
              </fileSets>
            </configuration>
          </execution>

This will include all code coverage exec files that were generated during the build and product a file named "jacoco-merged.exec" as defined by the property "jacoco.merge.execution.data.file"

A report phase is run subsequent to generating the merged exec file to produce a rolled up view of the code coverage.  Thisi s done with the following:

          <execution>
            <id>merged-report-test</id>
            <phase>post-integration-test</phase>
            <goals>
              <goal>report</goal>
            </goals>
            <configuration>
              <!-- Sets the path to the file which contains the execution data. -->
              <dataFile>${jacoco.merge.execution.data.file}</dataFile>
              <!-- Sets the output directory for the code coverage report. -->
              <outputDirectory>${project.reporting.outputDirectory}/mrg/jacoco-merge</outputDirectory>
            </configuration>
          </execution>

The final process run involves packaging up the code coverage results so they can be sent to the data warehouse.  Merged coverage results in the data warehouse contain the merged exec file and a zipped up file that contains all the results of the generated merged report from the "${project.reporting.outputDirectory}/mrg/jacoco-merge" folder.  These are generated with the following configuration:

      <plugin>
        <artifactId>maven-resources-plugin</artifactId>
        <version>2.7</version>
        <executions>
          <!-- Copy the rendered views to the rendered_views dir -->
          <execution>
            <id>copy-resources</id>
            <!-- here the phase you need -->
            <phase>verify</phase>
            <goals>
              <goal>copy-resources</goal>
            </goals>
            <configuration>
              <outputDirectory>${basedir}/rendered_views</outputDirectory>
              <resources>         
                <resource>
                  <!-- just grab the merged result view -->
                  <directory>${project.build.directory}/site/mrg</directory>
                  <filtering>true</filtering>
                </resource>
              </resources>              
            </configuration>            
          </execution>
        </executions>
      </plugin>

      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>2.2-beta-5</version>
        <configuration>
          <descriptors>
            <descriptor>zip.xml</descriptor>
          </descriptors>
          <fileSets>
            <includes>
              <include>${basedir}/rendered_views</include>
            </includes>
          </fileSets>

        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id> <!-- this is used for inheritance merges -->
            <phase>verify</phase>
            <goals>
              <goal>single</goal> <!-- goals == mojos -->
            </goals>
          </execution>
        </executions>
      </plugin>

NOTE that the make-assembly goal uses the file zip.xml to define where the rendered view zip file is to be placed.

Once the merge has been completed the post_build hook defined in the solano.yml file will be responsible for uploading the merged results to the data warehouse using the "upload_jacoco_merged_data" script.  This script is responsible for calculating the overall code coverage percentage based on data contained in the specified "jacoco.xml" file that is generated during the merged report phase.
