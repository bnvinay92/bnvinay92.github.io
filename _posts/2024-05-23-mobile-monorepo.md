# Kotlin Mobile Monorepo

We recently transitioned from a multi-repo to a monorepo for our mobile applications. This post will go over why and how we did this.

------

## Why Transition to a Monorepo?

### 1. Reduce Toil

Our team of iOS and Android developers previously managed three separate repositories: Android, iOS, and KMP (Kotlin Multiplatform). The KMP code streamed view state objects representing the view tree of the screen that platforms mapped to our in-house design library tokens (implemented with love using Compose and SwiftUI). This multi-repo setup involved a cumbersome workflow for landing the simplest of changes:

- Developers make changes in the KMP repo and publish artifacts locally
- They updated platform repos to point to the local artifact for testing.
- After raising and merging the PR on KMP, they released artifacts to our Artifactory instance.
- Finally, they raised PRs on platform repos to update artifact versions and made necessary changes.

If there were bugs or changes requested from review it only gets worse. We needed to eliminate the ceremony of multiple builds and reviews around landing a single change.

### 2. Shorter Lead Times

As outlined above, KMP changes had to go through several steps before landing on platform repos. There was no single commit we could revert confidently. This delay increased our time to release and hotfix, which is crucial during major outages. By consolidating into a monorepo, KMP changes land on platforms immediately, reducing lead times and improving our ability to respond quickly to issues.

### 3. Conflict Management

The probability of conflicts (both git and logical) scales exponentially with team size. With sixteen developers landing changes daily, it's a significant risk that's only multiplied by long lead times. By using a monorepo, we ensure that changes are integrated more frequently, reducing the chances of conflicts and making it easier to resolve them when they occur.

### 4. Improved Locality

While the transition to a monorepo didn't change much for iOS locality, Android developers experienced significant benefits. The unified repo enhances refactoring, debugging, and IDE support across Android/KMP projects, making the development process smoother and more efficient.

## What did we do

### Migrate the Repos

First, we created a new repo and moved all the code from the existing repos into it. The git subtree command proved very useful:

```
# In the new repo add current multirepos as remotes then:
git subtree pull --prefix=<kmp-directory-name> <kmp-repo-remote> <kmp-repo-main-branch>
git subtree pull --prefix=<android-directory-name> <android-repo-remote> <android-repo-main-branch>
git subtree pull --prefix=<ios-directory-name> <ios-repo-remote> <ios-repo-main-branch>
```

With this each repo will be pulled into the new repository as a subdirectory with the given name, and the history of the subdirectories will be preserved.
You can use the same command to transfer over open pull requests by changing the branch name.

### Gradle Composite Build

We used Gradle's composite build feature to combine android and KMP building into a single step, simplifying the build process. Here is an example of how this is done in the android `settings.gradle.kts`:

```kotlin
includeBuild("../your-kmp-gradle-project") {
    dependencySubstitution {
        substitute(module("com.#careem.rides.engine:rides-di"))
            .using(project(":rides:di"))
    }
}
```

We could have merged the two into a single Gradle project but this would have slowed down ios developers with android build configuration. It's also a big change that makes it harder to revert if things go wrong. We did not want to duplicate gradle-wrapper.properties files though since we point to an in house gradle distribution so we symlinked them. FYI, this won't work on Windows machines.

### Kotlin Native compilation filters

Though we couldn't use Gradle composite builds for iOS, we configured the Kotlin Multiplatform plugin to compile only necessary targets for our XCFramework. By default, it compiles three architectures, so this reduced build times by 65%.

```kotlin
extensions.configure(KotlinMultiplatformExtension::class) {
    val targets = this.targets
        .mapNotNull { it as? KotlinNativeTarget }
        .filter(useBuildFlagsIfAny) // Reads properties in the local.properties file
    val xcFramework = XCFramework(name)

    targets.forEach {
        it.binaries.framework(name) {
            this.configure()
            xcFramework.add(this)
        }
    }
}
```

### CI Workflow Optimization

Running all tests and builds for every change would be too expensive and time-consuming so we optimized our CI workflows to be conditional based on the files that have changed. One way to structure this would be to run a workflow that outputs jvm artifacts and the XCframework, and then fork into platform workflows based on the diffs. This approach doesn't exploit the fact that android workflows don't need to run on m1 machines, and that kotlin native compilation takes longer. Instead, we decided to fork the workflows from the start which lets use use a linux machine for the android workflow, and an m1 machine only for ios. KMP tests run on the linux machine right before android to sync up pipeline finish times. This also saves time by avoiding the deployment and retrieval of artifacts.

```
    android : Build KMP jvm jars --> Run KMP jvm tests and Sonar if KMP files changed --> Build APK --> Run android tests --> Run lint and sonar if android files changed
    
    
    ios     : Build KMP XCframework --> Run KMP iosSimulatorArm64 tests if KMP files changed --> Build Swift package --> Run ios tests --> Run ios lint and sonar if ios files changed
```

### Pull request labels

While visibility is important, we didn't want to clutter the PR view with too many PRs. We decided to write a Github action to label PRs based on the files that have changed. Developers can then filter PRs based on the labels. Feel free to steal the script:

```yaml
name: Label PRs

on:
  pull_request:
    branches:
      - main
    types: [opened, synchronize, reopened]

jobs:
  evaluate:
    permissions:
      contents: read
      pull-requests: write
    runs-on: ubuntu-latest
    steps:
      - name: Update PR Labels
        uses: actions/github-script@v6
        with:
          script: |
            const prNumber = context.payload.pull_request.number;
            const owner = context.repo.owner;
            const repo = context.repo.repo;

            console.log(`Processing PR #${prNumber} in ${repo} by ${owner}`);

            async function getCurrentLabels() {
              const response = await github.rest.issues.listLabelsOnIssue({
                owner,
                repo,
                issue_number: prNumber,
              });
              const labels = response.data.map(label => label.name);
              console.log("Current labels:", labels);
              return labels;
            }

            async function addLabelIfNeeded(pathPrefixes, label, currentLabels) {
              const files = await github.rest.pulls.listFiles({
                owner,
                repo,
                pull_number: prNumber
              });
              const filePaths = files.data.map(file => file.filename);
              console.log(`Files changed: ${filePaths.join(', ')}`);
              const isRelevant = filePaths.some(filePath => pathPrefixes.some(prefix => filePath.startsWith(prefix)));
              const needsLabel = isRelevant && !currentLabels.includes(label);
              console.log(`Checking for ${label} label addition, relevant: ${isRelevant}, needed: ${needsLabel}`);
              if (needsLabel) {
                await github.rest.issues.addLabels({
                  owner,
                  repo,
                  issue_number: prNumber,
                  labels: [label]
                }).catch(error => console.log(`Failed to add label: ${label}`, error));
              }
            }

            async function removeLabelIfNotNeeded(pathPrefixes, label, currentLabels) {
              const files = await github.rest.pulls.listFiles({
                owner,
                repo,
                pull_number: prNumber
              });
              const filePaths = files.data.map(file => file.filename);
              const isRelevant = filePaths.some(filePath => pathPrefixes.some(prefix => filePath.startsWith(prefix)));
              const needsRemoval = !isRelevant && currentLabels.includes(label);
              console.log(`Checking for ${label} label removal, relevant: ${isRelevant}, needed: ${needsRemoval}`);
              if (needsRemoval) {
                await github.rest.issues.removeLabel({
                  owner,
                  repo,
                  issue_number: prNumber,
                  name: label
                }).catch(error => console.log(`Failed to remove label: ${label}`, error));
              }
            }

            async function run() {
              const currentLabels = await getCurrentLabels();
              await addLabelIfNeeded(['android/', 'kmp/'], 'Android', currentLabels);
              await addLabelIfNeeded(['ios/', 'kmp/'], 'iOS', currentLabels);
              await removeLabelIfNotNeeded(['android/', 'kmp/'], 'Android', currentLabels);
              await removeLabelIfNotNeeded(['ios/', 'kmp/'], 'iOS', currentLabels);
            }

            run();
```

## Conclusion

By transitioning to a monorepo, we have streamlined our development process, reduced lead times, and improved overall efficiency. This change allows us to focus more on delivering high-quality features and less on managing the complexities of a multi-repo setup. The benefits in terms of reduced toil, shorter lead times, conflict management, and improved locality have made a significant positive impact on our workflow.

