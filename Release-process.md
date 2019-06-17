# Overview of the release process:

* Part 1: Pre release tasks
* Part 2: Release to Artifactory
* Part 3: Promote to Bintray and Maven Central
  * Part 3.1: From Artifactory to Bintray
  * Part 3.2: From Bintray to Maven Central
* Part 4: Post release tasks
* Part 5: Rolling back a release

# Part 1: Pre release tasks

1.1 Get latest changes from the upstream repository of the branch being released.

1.2 Check if `build.gradle` refers to any snapshot/milestone dependencies.

1.3 Disable the `Gradle` build plan in Bamboo:

<img alt="1-disable-build-plan" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/1-disable-build-plan.png">

# Part 2: Release to Artifactory

2.1 Run a build of the `Gradle` plan in Bamboo.

2.2 Go to `Artifactory build info` and check artifacts:

<img alt="2-1-check-build-info" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/2-1-check-build-info.png">

<img alt="2-2-check-artifacts" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/2-2-check-artifacts.png">

2.3 Go to `Artifactory Release & Promotion` tab:

<img alt="3-artifactory-build-release.png" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/3-artifactory-build-release.png">

2.3.1 Fill in the form with release version and VCS configuration then hit `Build and Release to Artifactory`. *NB:* Uncheck the `Use Release Branch` in the previous screenshot.

2.3.2 Check uploaded jars in http://repo.spring.io/libs-staging-local/ and do a smoke test with the staged version (check the integrity of the artifacts to see if jars are not corrupted or empty, etc).

2.4 Promote the release to Artifactory:

<img alt="4-artifactory-promote" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/4-artifactory-promote.png">

**NOTE:** For "Target promotion repository" field, select:

* `libs-milestone-local` for milestones
* `libs-release-local` for releases

2.5 Go to github and check if the release branch/tag are created:

<img alt="5-github" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/5-github.png">

2.6 Check uploaded jars in http://repo.spring.io/libs-release-local/

**Note: The process ends here for milestone releases**

# Part 3: Promote to Bintray and Maven Central

### Part 3.1 From Artifactory to Bintray

3.1.1 Go to https://repo.spring.io and login as `buildmaster`.

3.1.2 Check if published modules are as expected:

<img alt="6-published-modules.png" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/6-published-modules.png">

3.1.3 Go to `General build info` and click `Distribute`:

<img alt="7-distribute" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/7-distribute.png">

First, select "spring-distributions" in "Distribution Repository" do a "Dry run". There might be 3 errors about "schema.zip", "docs.zip" and "dist.zip" because they do not match any rule for artifacts with type Maven. These 3 errors can be safely ignored:

<img alt="8-errors" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/8-errors.png">

**Any other error should be carefully analyzed!**

Then click "Distribute". Artifacts should now be promoted to Bintray.

### Part 3.2 From Bintray to Maven Central

3.2.1 Go to https://bintray.com and login as `spring-operator`.

3.2.2 Select the corresponding release from the home page:

<img alt="9-bintray" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/9-bintray.png">

3.2.3 Go to `Maven Central` tab:

<img alt="10-sync" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/10-sync.png">

3.2.4 Fill in the form with the credentials of the "oss.sonatype.org" account (`springsource`) from LastPass vault (since we are pushing to "oss.sonatype.org") and hit "Sync".

3.2.5 Go to https://oss.sonatype.org and login with `springsource` account.

3.2.6 In `Staging Repositories` tab, select the staged release and check its content/activity:

<img alt="11-sonatype" src="https://raw.githubusercontent.com/wiki/spring-projects/spring-batch/images/release-process/11-sonatype.png">

After a few minutes, the staging repository will be **automatically** closed and the release will be pushed to Maven Central repository. Once the staging repository is closed, an information message will appear: `Staging repository has been released and can be safely dropped`. It is then safe to drop the repository.

# Part 4: Post release tasks

4.1 Upload docs to the docs server.

4.2 Reactivate the `Gradle` build plan in Bamboo.

4.3 Generate release notes in JIRA, release the version and create next version.

4.4 Update project page on Sagan with latest release/snapshot versions.

4.5 Write release announcement blog post.

4.6 Tweet about the release (make sure to cc `@SpringCentral` until `@SpringBatch` handle is ready).

4.7 Write email to the team.

4.8 Update `build.gradle` with snapshot versions of Spring Projects (Framework, AMQP, DATA, Integration and Kafka). This is to ensure the CI server builds against the latest snapshots and avoid last minute surprises during the next release.

# Part 5: Rolling Back a release

If anything goes wrong before promoting to Maven Central:

1. Delete version from Bintray.
2. Delete version from Artifactory.
3. Delete release branch/tag from git locally and remotely.
4. Revert the version in `gradle.properties` and push the change to the upstream repository.
5. Restart the release process.