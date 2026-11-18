# sbt-kubuszok

An sbt plugin that bundles and auto-configures common build settings for Scala projects using sbt-projectmatrix:
git-based versioning, publishing to Maven Central, formatting, coverage and cross-building.

## Installation

Add to `project/plugins.sbt`:

```scala
addSbtPlugin("com.kubuszok" % "sbt-kubuszok" % "0.2.5")
```

Pick the plugin version that matches your sbt version, and bump them **together**:

| sbt (`project/build.properties`) | sbt-kubuszok | JDK |
|---|---|---|
| 1.11.0+ (needed for the built-in Central publishing used by `ci-release`) | 0.2.4+ | 11+ |
| 2.0.x | 0.2.3 | 17+ |
| 2.1.x | 0.2.4+ | 17+ |

This single plugin brings in and configures the following plugins. Do not add them to `plugins.sbt` again:

- [sbt-git](https://github.com/sbt/sbt-git) (git-based versioning)
- [sbt-scalafmt](https://github.com/scalameta/sbt-scalafmt) (code formatting)
- [sbt-scoverage](https://github.com/scoverage/sbt-scoverage) (code coverage)
- [sbt-projectmatrix](https://github.com/sbt/sbt-projectmatrix) (cross-building; sbt 1.x only, built into sbt 2)
- [sbt-commandmatrix](https://github.com/indoorvivants/sbt-commandmatrix) (matrix configuration)
- [sbt-scalajs](https://github.com/scala-js/scala-js) (Scala.js)
- [sbt-scala-native](https://github.com/scala-native/scala-native) (Scala Native)
- [sbt-pgp](https://github.com/sbt/sbt-pgp) (artifact signing)
- [sbt-mima-plugin](https://github.com/lightbend/mima) (binary compatibility)
- [sbt-ide-settings](https://github.com/JetBrains/sbt-ide-settings) (IDE project filtering)
- [sbt-welcome](https://github.com/reibitto/sbt-welcome) (useful tasks display; sbt 1.x only, it has no sbt 2 build)

## What it provides

### KubuszokPlugin (auto-triggered)

Activates automatically for every project. The following are already configured, so do not set them in your build:

- **Git versioning**: `GitVersioning` with `git describe`.
  - A tag on HEAD gives the release version (`1.0.0`).
  - Otherwise the version is `<last-tag>-<n>-g<sha>-SNAPSHOT`, or `<sha>-SNAPSHOT` when there is no tag or the
    clone is shallow.
  - `isSnapshot` is true exactly when HEAD has no tag; uncommitted changes do not matter.
  - Do not set `version` yourself.
- **Publishing**:
  - `publishTo`: snapshots go to Maven Central Snapshots, releases go to sbt's `localStaging`.
  - Also sets `publishMavenStyle`, `pomIncludeRepository`, `versionScheme := Some("early-semver")` and
    `Test / publishArtifact := false`.
  - POM metadata (`organization`, `homepage`, `licenses`, `developers`, `scmInfo`, `description`) is up to you,
    since Central requires it. Keep the sources and doc JARs; Central rejects releases without a javadoc JAR.
- **`ci-release` command**: runs `+publishSigned` (every cross-built axis).
  - If HEAD has **any** tag, it then runs `sonaRelease` (uploads and releases the bundle).
  - It reads `SONATYPE_USERNAME` and `SONATYPE_PASSWORD` from the environment, and signs with the `gpg` key
    available on the machine.
- **`checkNoSnapshotDeps`**: runs before `publish`/`publishSigned`, and fails a release version that depends on any
  `-SNAPSHOT` module.
- **`projectType` setting**: controls publishing per project.
  - `ProjectType.NonPublished` is the **default**. It sets `publish / skip := true`.
  - `ProjectType.ScalaLibrary` is published. Set it on every library module.
  - `ProjectType.JarOnly` is published too. Use it for sbt plugins and Java/resource JARs; for a non-Scala JAR, also
    set `crossPaths := false` and `autoScalaLibrary := false`.
- **`scalafmtOnCompile`**: enabled in dev, disabled on CI (detected via the `CI=true` env var, exposed as `isCI`).
  CI should run `scalafmt --check`.
- **`foldVersion` helper**: pattern match on the Scala binary version:
  ```scala
  scalacOptions ++= foldVersion(scalaVersion.value)(
    for2_13 = Seq("-Ytasty-reader"),
    for3 = Seq("-Xcheck-macros")
  )
  ```
- **Resolver**: Maven Central Snapshots (`mavenCentralSnapshots`), added globally.

### KubuszokRootPlugin (manual trigger)

Enable it on the root/aggregation project only. The root must aggregate every published module, because
`ci-release` runs from there. On sbt 1.x it adds `WelcomePlugin` (`logo`, `usefulTasks`); on sbt 2 it adds nothing
else.

```scala
lazy val root = project
  .in(file("."))
  .enablePlugins(KubuszokRootPlugin)
  .settings(al.commandAliases(publishLocalForTestsFilter = Some((_, platform) => platform == "JVM")))
  .aggregate(core.projectRefs *)
```

### DevProperties

Configures IDE visibility for multi-platform/multi-version project matrix builds. Reads from `dev.properties`:

```scala
val dev = new DevProperties(
  scala213 = Some(versions.scala213),
  scala3 = Some(versions.scala3),
  platforms = List(VirtualAxis.jvm, VirtualAxis.js, VirtualAxis.native)
)

// Use in project matrix definitions:
lazy val myLib = projectMatrix
  .someVariations(versions.scalas, versions.platforms)(dev.only1VersionInIDE *)
```

`dev.properties` (per developer; commit a default and run `git update-index --assume-unchanged dev.properties` before
changing it locally):
```properties
# 2.13 or 3 (defaults to 3 when both are built)
ide.scala=3
# jvm, js or native (required when more than one platform is built)
ide.platform=jvm
# project-specific keys are fine too
log.cross-quotes=false
```

Read project-specific keys with `dev.prop(name)`. It returns the trimmed value, or `None` if the key is missing or
blank:

```scala
val logCrossQuotes: Boolean = dev.prop("log.cross-quotes").fold(!isCI)(_ == "true")
```

### Aliases

Derives CI/test/publish commands from `ProjectMatrix` instances grouped by role:

- **`published`**: compiled, tested, documented and published (included in `publishLocal` and `publishLocalForTests`)
- **`testOnly`**: compiled and tested, but not published
- **`compileOnly`**: only verified for compilation

These groups only drive the generated commands. Whether a module is published is decided by its `projectType`, so
keep the two in sync. Modules are deduplicated across groups, so accidental overlap won't produce duplicate tasks.

```scala
lazy val al = new Aliases(
  published = Seq(core, utils, integration),
  testOnly = Seq(tests),
  compileOnly = Seq(benchmarks)
)

// Commands for each (platform, scala) combination ("JVM" | "JS" | "Native", "2.13" | "3"):
al.ci("JVM", "3")       // clean ; coverage ; core3/compile ; ... ; core3/test ; ... ; coverageAggregate ; coverageOff ; core3/doc ; ...
al.test("JS", "2.13")   // coreJS/test ; utilsJS/test ; testsJS/test ; ...
al.release              // "ci-release" (the built-in command)
al.publishLocal("JVM", "2.13")
```

- `ci` runs `doc` for the `published` modules, because `ci-release` does too. This way a broken Scaladoc link fails CI
  instead of the release. Override `docInCi` to disable it.
- On sbt 2, the generated commands use `testFull` instead of `test`. sbt 2's `test` is incremental and cached, so a
  fresh CI run could otherwise report "No tests to run".

Register the commands as aliases (`ci-jvm-3`, `ci-js-2_13`, `test-native-3`, ..., plus `publish-local-for-tests`)
on the root project:

```scala
// sbt 1.x and sbt 2.x
.settings(al.commandAliases(publishLocalForTestsFilter = Some((_, platform) => platform == "JVM")))

// or, sbt 1.x only: the same aliases plus sbt-welcome's task list
usefulTasks := al.usefulTasks(
  publishLocalForTestsFilter = Some((_, platform) => platform == "JVM"),
  extra = Seq(
    UsefulTask("myCustomCommand", "Does something special").alias("custom")
  )
)
```

Do not add a `ci-release` alias: the command already exists.

Subclass for project-specific needs:

```scala
lazy val al = new Aliases(published = Seq(core, utils, tests)) {
  override def ci(platform: String, scalaBinary: String): String = {
    val base = super.ci(platform, scalaBinary)
    val mimaIds = projectIds(published, platform, scalaBinary)
    s"$base ; ${mimaIds.map(id => s"$id/mimaReportBinaryIssues").mkString(" ; ")}"
  }
}
```

## GitHub Actions

- **CI**: a matrix over `scala: ['2_13', '3']` × `platform: ['jvm', 'js', 'native']` running
  `sbt ci-${{ matrix.platform }}-${{ matrix.scala }}`, plus a `scalafmt --check` job.
- **Release**: [`.github/workflows/release.yml`](.github/workflows/release.yml) of this repository is the template.
  - Trigger it on `push` to `master`, on tags, and on `pull_request` for trusted authors only. PR snapshots let a
    change be tested in other repos before releasing. Bot and fork PRs have no secrets, so guard the job with
    `if: github.event_name != 'pull_request' || contains(fromJSON('["OWNER", "MEMBER", "COLLABORATOR"]'), github.event.pull_request.author_association)`.
  - Use `actions/checkout` with `fetch-depth: 0`.
  - Import the GPG key with `crazy-max/ghaction-import-gpg` (`GPG_PRIVATE_KEY`, `GPG_PASSPHRASE`).
  - Run `sbt ci-release` with `SONATYPE_USERNAME`/`SONATYPE_PASSWORD` in `env`.
- **Releasing**: push a bare version tag (`1.2.3`) on a master commit.

## Self-bootstrapping

The plugin uses itself for its own build via `project/build.sbt`:

```scala
Compile / unmanagedSourceDirectories +=
  baseDirectory.value / ".." / "src" / "main" / "scala"
```

This compiles the plugin source in the meta-build, so the same code is both published as an sbt plugin and used to
configure its own publishing.

## License

Apache 2.0 - see [LICENSE](LICENSE).
