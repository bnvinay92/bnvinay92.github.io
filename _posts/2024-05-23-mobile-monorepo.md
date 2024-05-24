# Kotlin Mobile Monorepo

We recently migrated from a multi-repo to a monorepo for our ios and android mobile application. This post will go over why and how we did this.

------

## Why Migrate to a Monorepo?

#### Context

Our team of iOS and android developers previously managed three separate repositories: android, iOS, and KMP (Kotlin Multiplatform). At a high level, the KMP code streams view state objects representing the view tree of the screen that platforms render by mapping to our in-house design library tokens (implemented with love using Compose and SwiftUI). Some pain points with this multi-repo setup were:

### Toil

Most contributions touch KMP code due to our architecture. To even land the simplest change the workflow was:

- Developers make changes in the KMP repo and publish artifacts locally
- We update platform repos to point to the local artifact for testing
- After raising and merging the PR on KMP, we release artifacts to our internal artifactory
- Finally, we raise PRs on platform repos that update artifact versions and make necessary changes to consume the new artifact
- If there were bugs or changes requested from review, or if conflicting changes were made before you landed your changes you loop back from the start

![multirepo workflow](/assets/multirepo_workflow.png)

We needed to eliminate the ceremony of multiple builds and reviews around landing a single change.

### Long Lead Times

As outlined above, KMP changes had to go through several steps before landing on platform repos. This delay increased our time to release and hotfix, which is crucial during major outages. There isn't a single commit you can confidently revert.
By consolidating into a monorepo, KMP changes land on platforms immediately, reducing the time to detect and resolve issues.

![conflict graph](/assets/repo_conflicts.png)

The probability of logical conflicts scales exponentially with team size. With sixteen developers landing changes daily, it becomes a significant risk that's only multiplied by long lead times. A monorepo ensures that changes land sooner, reducing the time to detection and probability of conflicts.

### Lack of locality

A multirepo setup degrades the experience of refactoring, debugging, and IDE support across android/KMP projects. IDE switching adds up overtime. Unfortunately, switching between Xcode and Android Studio would not be solved by a monorepo.

### Alternatives

Monorepos don't come for free:
 
- Requires teamwork: Developers may not be familiar with both platforms and might require to pair to land certain changes. You might also see more git conflicts if developers don't coordinate their work. Compared to a multirepo though, you force that coordination sooner rather than later which is a good thing imo
- CI build times and cost: This needs mitigation through optimization. More on this later
- Repo size: This is not a concern for our project

But the alternatives aren't appealing:

- Submodules: Just no, you'd be replacing a version bump with a submodule update. We'd be maintaining the complexity of our workflow and dealing with the cons of both multirepos and monorepos
- Automating platform PRs: This would be a band-aid solution that doesn't address the root cause of our problems. Platform PRs would still require changes to be made, and we'd still have to deal with long lead times and conflicts.

## What did we do

### Migrate the Repos

First, we created a new repo and moved all the code from the existing repos into it. The git subtree command proved very useful:

```bash
# In the new repository
git subtree pull --prefix=<kmp-directory-name> <kmp-repo-remote> <kmp-repo-main-branch>
git subtree pull --prefix=<android-directory-name> <android-repo-remote> <android-repo-main-branch>
git subtree pull --prefix=<ios-directory-name> <ios-repo-remote> <ios-repo-main-branch>
```

With this each repo will be pulled into the new repository as a subdirectory with the given name, and the history of the subdirectories will be preserved.
You can use the same command to transfer over open pull requests by changing the branch name.

### Compose Gradle builds

We used Gradle's [composite build](https://docs.gradle.org/current/userguide/composite_builds.html) feature to combine android and KMP building into a single step, simplifying the build process. Here is an example of how this is done in the android `settings.gradle.kts`:

```kotlin
includeBuild("../your-kmp-gradle-project") {
    dependencySubstitution {
        substitute(module("com.careem.rides.engine:rides-di"))
            .using(project(":rides:di"))
    }
}
```

We could have merged the two into a single Gradle project but this would have slowed down ios builds with android build configuration. It's also a big change that makes it harder to revert if things go wrong.

We did not want to duplicate our custom build logic so we extracted them to a separate gradle project. This approach also means you have two separate gradle wrappers with their own gradle-wrapper.properties files which we also did not want to duplicate (we point to an in house gradle distribution), so we symlinked them. FYI, symlinking won't work on Windows machines.

### Kotlin Native compilation filters

Though we couldn't use Gradle composite builds for iOS, we configured the Kotlin Multiplatform plugin to compile only necessary targets for our XCFramework. By default, it compiles three architectures so this little snippet reduced XCFramework build times by 65%:

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

On CI we target only the simulator architecture for testing since we have no architecture specific source set.

### CI Workflow Optimization

Running all tests and builds for every change would be too expensive and time-consuming, so we optimized our CI workflows to be conditional based on the files that have changed.

One way to structure this would be to run a workflow that outputs JVM artifacts and the XCFramework, and then fork into platform workflows based on the diffs. This approach doesn't exploit the fact that android workflows don't need to run on M1 machines, and that Kotlin Native compilation takes longer.
Instead, we decided to fork the workflows from the start, which lets us use a Linux machine for the android workflow, and an M1 machine only for iOS. KMP tests run on the Linux machine right before android to sync up pipeline finish times. This also saves time by avoiding the deployment and retrieval of artifacts in the pipeline. Each platform pipeline looks something like this:

![ci](/assets/ci.png)

We were consuming ~75 bitrise credits per pull request on our KMP repo, and we're at ~85 credits per PR on the monorepo. This was surprising at first but given that the bulk of the work time was spent in XCFramework compilation and the cost of M1 machines it makes sense.
Granted we're missing a verification or two that still needs to be moved over and will add some time, it's a small price to pay for the benefits we've gained.

### Pull request labels

While visibility is important, we didn't want to clutter the PR view with too many PRs. We decided to write a Github action to label PRs based on the files that have changed. Developers can then filter PRs based on the labels. Feel free to steal the script from [here](https://github.com/bnvinay92/pr-labeler/blob/main/.github/workflows/pr-labels.yml).

## Conclusion

It's been a month since we transitioned to a monorepo, and it feels like we've removed a thorn we had become numb to. You don't notice the daily pain of bouncing contributions to multiple repos until it's gone. This change has significantly reduced toil, shortened lead times, improved conflict management, and enhanced our overall development process.

