# ScriptRunner Scripts Build Guide

This guide explains how to build the **ScriptRunner scripts repository** (Groovy scripts for Adaptavist ScriptRunner on Jira and Confluence) on any machine: Linux, macOS (Intel or Apple Silicon), or Windows, on x86_64 or ARM64.

Unless stated otherwise, paths are relative to the repository root (the folder that contains this file and `pom.xml`). Replace placeholders such as `<SETTINGS_DIR>`, `<MAVEN_INSTALL_DIR>` or `<M2_REPO>` with locations of your choice. `<M2_REPO>` is your local Maven repository: `~/.m2/repository` on Linux/macOS, `%USERPROFILE%\.m2\repository` on Windows.

---

## 1. What you are building

| Item | Value |
|------|-------|
| Host products | Jira Server/Data Center **8.10.1** (`jira.version`), Jira Software (Agile) 8.10.1 (`agile.version`), Jira Service Desk API 4.0.2-REL-0002 (`servicedesk.version`), Confluence **7.7.3** (`confluence.version`), Insight 8.4.4 (`insight.version`), all in `pom.xml` |
| ScriptRunner | `com.onresolve.jira.groovy:groovyrunner` **6.9.1** (`scriptrunner.version` in `pom.xml`) |
| Groovy | **2.5.13** (`groovy.version` in `pom.xml`) |
| Build tool | Apache Maven 3 (no wrapper in the repository; verified with Maven **3.9.9**) |
| Required JDK | **11** recommended. `mvn compile` was verified on JDK 8, 11, 17, 21 and 25. The optional script compile check (section 5.2) needs JDK **9 to 16** (verified on 11); Groovy 2.5.13 fails on JDK 17+ and JDK 8 has no `@argfile` support |
| Scripts | 231 `.groovy` files under `src/main/jira`, `src/main/jira_cloud` and `src/main/confluence` |
| Outputs | `mvn compile`: none (see 2.1). Optional compile check: `.class` files in `target/scripts-classes/` |

The repository is the Adaptavist "scripts repository" template: the `pom.xml` exists so that an IDE (IntelliJ IDEA) can resolve the Jira, Confluence and ScriptRunner APIs for code completion. The scripts themselves are deployed as source files to a ScriptRunner script root (section 7); nothing is packaged.

---

## 2. Things to know before building

### 2.1 `mvn compile` does not compile the scripts

The scripts live in `src/main/jira`, `src/main/jira_cloud` and `src/main/confluence`, not in a Maven source folder, and the `pom.xml` has no Groovy compiler plugin. `mvn compile` therefore resolves all declared dependencies, prints `No sources to compile` and ends with `BUILD SUCCESS`. Its real value is proving that the full Jira/Confluence/ScriptRunner classpath can be resolved.

To actually compile every script against that classpath, use the optional check in section 5.2 (it uses only Maven goals and the Groovy compiler that is already a dependency; no file in the repository is changed).

### 2.2 The repositories declared in `pom.xml` no longer exist

| Declared in `pom.xml` | Status | Replacement |
|------|-------|-------------|
| `adaptavist.public` = `https://nexus.adaptavist.com/content/repositories/external/` | Host does not resolve (DNS) | `https://nexus.adaptavist.net/content/repositories/external/` (hosts `groovyrunner` 6.9.1) |
| `riada` = `https://repo.riada.io/repository/riada-repo/` | Host does not resolve (DNS) | `https://packages.atlassian.com/maven-external/` (hosts `insight` 8.4.4 `api` jar) |

All Atlassian artifacts (Jira, Confluence, Jira Software, Service Desk, Insight, Atlassian libraries) come from **`https://packages.atlassian.com/maven-external/`**. Everything else comes from **Maven Central** (`https://repo.maven.apache.org/maven2`). The build needs exactly these three repositories to be reachable (whitelist them on a proxy or firewall):

- `https://packages.atlassian.com/maven-external/`
- `https://nexus.adaptavist.net/content/repositories/external/`
- `https://repo.maven.apache.org/maven2/`

No credentials are needed. Since `pom.xml` must not be edited, the replacements are configured in a Maven `settings.xml` outside the repository (section 4.1).

### 2.3 Three legacy Sun jars are not in any public repository

`jira-core` depends on `jta:jta:1.0.1`, `jndi:jndi:1.2.1` and `javax.activation:activation:1.0.2`. These jars cannot be downloaded from any public Maven repository (only their POMs are on Maven Central), so the build fails with `Could not find artifact jta:jta:jar:1.0.1` (section 8). Atlassian ships them in the local repository bundled with the **Atlassian Plugin SDK**; section 4.2 copies them from the SDK archive into `<M2_REPO>`. You do not need to install the SDK.

### 2.4 Dependency resolution is slow without the parallel collector

The dependency tree is large (about 600 jars, about 330 MB in `<M2_REPO>`) and `packages.atlassian.com` answers slowly per request. With Maven's default dependency collector the first resolution ran for more than 25 minutes without finishing. With the breadth-first collector (`-Daether.dependencyCollector.impl=bf`, Maven 3.9+) the same first build took about 3 minutes. All commands below use it.

### 2.5 Harmless messages

- `[WARNING] The POM for com.onresolve.jira.groovy:groovyrunner:jar:6.9.1 is invalid, transitive dependencies (if any) will not be available` and many `[ERROR] 'dependencies.dependency.version' for ... is missing` lines: these refer to dependency POMs (groovyrunner and some Atlassian POMs reference parents that are not public). They do not fail the build.
- `WARNING: An illegal reflective access operation has occurred` during the compile check: Groovy 2.5 on JDK 11, harmless.

### 2.6 Two scripts have a broken import

`src/main/jira/listeners/set_field_based_on/PriorityChange.groovy` and `src/main/jira/listeners/set_field_based_on/LinkedIssuesWithCondition.groovy` contain `import JiraUtilHelper`, but that class is declared in package `jira` (`src/main/jira/JiraUtilHelper.groovy`). The compile check (5.2) therefore reports exactly 2 errors. Changing the line to `import jira.JiraUtilHelper` in both files makes all 231 scripts compile (verified on a copy). The repository is left unchanged; section 5.2 shows how to check the other 229 scripts.

---

## 3. Install the JDK (11 recommended)

Any OpenJDK 11 build works (Eclipse Temurin, Azul Zulu, Amazon Corretto, Microsoft, Liberica, etc.), with native x86_64 and ARM64 builds including Apple Silicon. JDK 11 is also what Jira 8.10 runs on.

| Platform | Command |
|----------|---------|
| SDKMAN (Linux, macOS, WSL) | `sdk install java 11.<x>-tem` (pick a version from `sdk list java`) |
| Debian / Ubuntu | `sudo apt install openjdk-11-jdk` |
| Fedora / RHEL | `sudo dnf install java-11-openjdk-devel` |
| Arch | `sudo pacman -S jdk11-openjdk` |
| macOS (Homebrew) | `brew install --cask temurin@11` |
| Windows (winget) | `winget install EclipseAdoptium.Temurin.11.JDK` |
| Windows (Chocolatey) | `choco install temurin11` |
| Manual | https://adoptium.net/temurin/releases/?version=11 |

Verify:

```bash
java -version    # must print version "11.x"
```

---

## 4. Install and configure Maven

| Option | Command |
|--------|---------|
| SDKMAN (Linux, macOS, WSL) | `sdk install maven 3.9.9` |
| Homebrew (macOS) | `brew install maven` |
| Windows (winget / Chocolatey) | `winget install Apache.Maven` / `choco install maven` |
| Linux distributions | `apt` / `dnf` / `pacman` packages may ship Maven older than 3.9; check `mvn -v` |
| Manual (all platforms) | Download https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.tar.gz (or `.zip`) and extract it to `<MAVEN_INSTALL_DIR>` |

Any Maven **3.9.x** works; the `bf` collector option (2.4) needs 3.9 or newer.

**Linux / macOS / WSL:**

```bash
export JAVA_HOME=<path-to-jdk11>
export PATH="$JAVA_HOME/bin:<MAVEN_INSTALL_DIR>/apache-maven-3.9.9/bin:$PATH"
```

**Windows PowerShell:**

```powershell
$env:JAVA_HOME = "<path-to-jdk11>"
$env:Path = "$env:JAVA_HOME\bin;<MAVEN_INSTALL_DIR>\apache-maven-3.9.9\bin;$env:Path"
```

**Windows cmd.exe:**

```bat
set JAVA_HOME=<path-to-jdk11>
set PATH=%JAVA_HOME%\bin;<MAVEN_INSTALL_DIR>\apache-maven-3.9.9\bin;%PATH%
```

Verify:

```bash
mvn -v    # Apache Maven 3.9.x, Java version: 11.x
```

### 4.1 Create `settings.xml` (repository replacements)

Save the following as `<SETTINGS_DIR>/settings.xml`, outside the repository. (You may also merge it into your `~/.m2/settings.xml`; then drop the `-s` option below.)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <mirrors>
    <!-- nexus.adaptavist.com no longer exists; Adaptavist now serves the same repository from nexus.adaptavist.net -->
    <mirror>
      <id>adaptavist-net</id>
      <mirrorOf>adaptavist.public</mirrorOf>
      <url>https://nexus.adaptavist.net/content/repositories/external/</url>
    </mirror>
    <!-- repo.riada.io no longer exists; the Insight API jar is on Atlassian's public repository -->
    <mirror>
      <id>atlassian-for-riada</id>
      <mirrorOf>riada</mirrorOf>
      <url>https://packages.atlassian.com/maven-external/</url>
    </mirror>
  </mirrors>
  <profiles>
    <profile>
      <id>atlassian-public</id>
      <repositories>
        <repository>
          <id>atlassian-public</id>
          <url>https://packages.atlassian.com/maven-external/</url>
        </repository>
      </repositories>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>atlassian-public</activeProfile>
  </activeProfiles>
</settings>
```

### 4.2 Add the three legacy jars to the local Maven repository

Download the Atlassian Plugin SDK archive (about 84 MB; SHA-256 `b0401ed5e89ff6a5ef20ebbc45f6783169e010ee720a353be20e02addf392776`) to any temporary folder and extract only the three artifacts into `<M2_REPO>`:

**Linux / macOS / WSL:**

```bash
mkdir -p ~/.m2/repository
curl -fL -o atlassian-plugin-sdk-9.1.1.tar.gz \
  https://packages.atlassian.com/maven-public/com/atlassian/amps/atlassian-plugin-sdk/9.1.1/atlassian-plugin-sdk-9.1.1.tar.gz
tar -xzf atlassian-plugin-sdk-9.1.1.tar.gz --strip-components=2 -C ~/.m2/repository \
  atlassian-plugin-sdk-9.1.1/repository/jta/jta/1.0.1 \
  atlassian-plugin-sdk-9.1.1/repository/jndi/jndi/1.2.1 \
  atlassian-plugin-sdk-9.1.1/repository/javax/activation/activation/1.0.2
```

**Windows PowerShell** (Windows 10 1803+ ships `curl.exe` and `tar.exe`; not verified):

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.m2\repository" | Out-Null
curl.exe -fL -o atlassian-plugin-sdk-9.1.1.tar.gz https://packages.atlassian.com/maven-public/com/atlassian/amps/atlassian-plugin-sdk/9.1.1/atlassian-plugin-sdk-9.1.1.tar.gz
tar.exe -xzf atlassian-plugin-sdk-9.1.1.tar.gz --strip-components=2 -C "$env:USERPROFILE\.m2\repository" atlassian-plugin-sdk-9.1.1/repository/jta/jta/1.0.1 atlassian-plugin-sdk-9.1.1/repository/jndi/jndi/1.2.1 atlassian-plugin-sdk-9.1.1/repository/javax/activation/activation/1.0.2
```

Afterwards `<M2_REPO>` contains `jta/jta/1.0.1/jta-1.0.1.jar`, `jndi/jndi/1.2.1/jndi-1.2.1.jar` and `javax/activation/activation/1.0.2/activation-1.0.2.jar`. The archive can then be deleted. (Verified with GNU tar on Linux; macOS and Windows `tar` are bsdtar, which supports the same options but was not tested.)

If a previous failed build already left `*.lastUpdated` files for these artifacts, that is fine: Maven uses the jar once it exists.

---

## 5. Build

### 5.1 Maven build

From the repository root:

```bash
mvn -B -s <SETTINGS_DIR>/settings.xml -Daether.dependencyCollector.impl=bf compile
```

In PowerShell quote the `-D` option: `mvn -B -s <SETTINGS_DIR>\settings.xml "-Daether.dependencyCollector.impl=bf" compile` (not verified).

Expected end of the output:

```
[INFO] No sources to compile
[INFO] BUILD SUCCESS
```

The first run downloads about 330 MB into `<M2_REPO>` and took about 3 minutes; later runs take a few seconds. No artifact is produced (2.1); `target/` is not even created.

### 5.2 Optional: compile every script against the declared classpath

This uses `maven-dependency-plugin` (downloaded automatically) to collect the classpath and the Groovy 2.5.13 compiler that is part of that classpath. It needs **JDK 9 to 16** (verified on 11). Everything is written under `target/`.

**Linux / macOS / WSL:**

```bash
MVN="mvn -B -s <SETTINGS_DIR>/settings.xml -Daether.dependencyCollector.impl=bf"
$MVN dependency:copy-dependencies -DincludeScope=compile -DexcludeArtifactIds=groovy-all -DoutputDirectory=target/lib
$MVN dependency:build-classpath -DincludeScope=compile -DexcludeArtifactIds=groovy-all \
     -Dmdep.prefix=target/lib -Dmdep.outputFile=target/classpath.txt
{ printf -- '-cp '; cat target/classpath.txt; } > target/groovyc.args
find src/main -name '*.groovy' > target/scripts.txt
java @target/groovyc.args org.codehaus.groovy.tools.FileSystemCompiler -d target/scripts-classes @target/scripts.txt
```

Notes on the options:

- `includeScope=compile` also includes `provided` dependencies (Jira, Confluence, Insight).
- `excludeArtifactIds=groovy-all` drops `groovy-all-2.4.15.jar`, which arrives transitively and would clash with Groovy 2.5.13.
- The classpath is built in Maven's order through a Java argument file (`@target/groovyc.args`) because the order matters: with `-cp "target/lib/*"` the JVM picks jars in directory order and Groovy fails with `java.lang.NoSuchMethodError: ... StatusLogger.getLogger()` (log4j version clash).
- `src/main` acts as the script root: every script's `package` matches its folder under `src/main`, so helper classes such as `jira.JiraUtilHelper` are found when all files are compiled together.

Expected result with the repository as published: the compiler stops with exactly these two errors (see 2.6) and writes no classes:

```
src/main/jira/listeners/set_field_based_on/PriorityChange.groovy: 3: unable to resolve class JiraUtilHelper
src/main/jira/listeners/set_field_based_on/LinkedIssuesWithCondition.groovy: 5: unable to resolve class JiraUtilHelper
2 errors
```

To compile the other 229 scripts:

```bash
grep -v -e PriorityChange.groovy -e LinkedIssuesWithCondition.groovy target/scripts.txt > target/scripts-ok.txt
java @target/groovyc.args org.codehaus.groovy.tools.FileSystemCompiler -d target/scripts-classes @target/scripts-ok.txt
```

This exits with code 0 and writes 965 `.class` files to `target/scripts-classes/` (about 10 seconds). With the import fixed in both files, all 231 scripts compile (987 classes). `target/lib` takes about 270 MB. The `.class` files are only a verification result: ScriptRunner compiles the `.groovy` sources itself at runtime.

**Windows PowerShell** (not verified; Maven writes `target\classpath.txt` with `;` separators and forward slashes from `-Dmdep.prefix`, so the argument file needs no escaping):

```powershell
$m = @("-B", "-s", "<SETTINGS_DIR>\settings.xml", "-Daether.dependencyCollector.impl=bf")
mvn @m dependency:copy-dependencies "-DincludeScope=compile" "-DexcludeArtifactIds=groovy-all" "-DoutputDirectory=target/lib"
mvn @m dependency:build-classpath "-DincludeScope=compile" "-DexcludeArtifactIds=groovy-all" "-Dmdep.prefix=target/lib" "-Dmdep.outputFile=target/classpath.txt"
"-cp " + (Get-Content target\classpath.txt -Raw).Trim() | Set-Content -Encoding ascii target\groovyc.args
Get-ChildItem -Recurse -Filter *.groovy src\main | ForEach-Object { (Resolve-Path -Relative $_.FullName).Replace('\','/') } | Set-Content -Encoding ascii target\scripts.txt
java "@target/groovyc.args" org.codehaus.groovy.tools.FileSystemCompiler -d target/scripts-classes "@target/scripts.txt"
```

Useful commands:

```bash
mvn -B -s <SETTINGS_DIR>/settings.xml -Daether.dependencyCollector.impl=bf dependency:tree   # show the resolved tree
mvn clean                                                                                   # delete target/
```

---

## 6. Build with Docker (no local JDK or Maven)

The official `maven:3.9.9-eclipse-temurin-11` image is multi-arch (amd64, arm64 and others). From the repository root, with the `settings.xml` from section 4.1 in `<SETTINGS_DIR>`:

```bash
docker run --rm -e MAVEN_OPTS=-Xmx1536m \
  -v "$PWD":/src -v "<SETTINGS_DIR>/settings.xml":/settings.xml:ro \
  -v scriptrunner-m2:/root/.m2 -w /src maven:3.9.9-eclipse-temurin-11 bash -c '
  set -e
  mkdir -p /root/.m2/repository
  curl -fsSL https://packages.atlassian.com/maven-public/com/atlassian/amps/atlassian-plugin-sdk/9.1.1/atlassian-plugin-sdk-9.1.1.tar.gz \
    | tar xz --strip-components=2 -C /root/.m2/repository \
        atlassian-plugin-sdk-9.1.1/repository/jta/jta/1.0.1 \
        atlassian-plugin-sdk-9.1.1/repository/jndi/jndi/1.2.1 \
        atlassian-plugin-sdk-9.1.1/repository/javax/activation/activation/1.0.2
  MVN="mvn -B -s /settings.xml -Daether.dependencyCollector.impl=bf"
  $MVN compile
  $MVN dependency:copy-dependencies -DincludeScope=compile -DexcludeArtifactIds=groovy-all -DoutputDirectory=target/lib
  $MVN dependency:build-classpath -DincludeScope=compile -DexcludeArtifactIds=groovy-all -Dmdep.prefix=target/lib -Dmdep.outputFile=target/classpath.txt
  { printf -- "-cp "; cat target/classpath.txt; } > target/groovyc.args
  find src/main -name "*.groovy" > target/scripts.txt
  java @target/groovyc.args org.codehaus.groovy.tools.FileSystemCompiler -d target/scripts-classes @target/scripts.txt
'
```

With a fresh cache this ran in about 3.5 minutes: the three Maven steps end with `BUILD SUCCESS` and the last step reports the two expected errors from section 5.2 (exit code 1). The named volume `scriptrunner-m2` keeps the downloaded dependencies between runs (`docker volume rm scriptrunner-m2` removes it). To compile the 229 other scripts afterwards (exit code 0, 965 classes):

```bash
docker run --rm -v "$PWD":/src -v scriptrunner-m2:/root/.m2 -w /src maven:3.9.9-eclipse-temurin-11 bash -c '
  grep -v -e PriorityChange.groovy -e LinkedIssuesWithCondition.groovy target/scripts.txt > target/scripts-ok.txt
  java @target/groovyc.args org.codehaus.groovy.tools.FileSystemCompiler -d target/scripts-classes @target/scripts-ok.txt'
```

On Windows PowerShell replace `"$PWD":/src` with `"${PWD}:/src"` and `"<SETTINGS_DIR>/settings.xml"` with the Windows path of the file (not verified). On Linux, files created in `target/` by the container are owned by `root`; remove them with `sudo` if needed. The Docker runs were verified on Linux x86_64 only.

---

## 7. Use (not verified: no Jira or Confluence instance was available)

The scripts are deployed as source files; ScriptRunner compiles them at runtime.

- **Script roots (Jira/Confluence Server and Data Center).** ScriptRunner looks for scripts in its script roots, by default `<jira-home>/scripts` (or `<confluence-home>/scripts`). Copy the folders under `src/main` (for example `src/main/jira`) into a script root so that the folder structure matches the `package` declarations (`package jira.listeners...` must be at `<script-root>/jira/listeners/...`). Shared helpers such as `jira/JiraUtilHelper.groovy` and `jira/ReportGenerator.groovy` are then importable from other scripts.
- **Script Console.** Administration > ScriptRunner > Console: paste a script (`src/main/jira/script_console/...`) or choose the File tab and enter its path relative to a script root.
- **Workflow functions, listeners, scripted fields, behaviours, jobs, escalation services.** In the corresponding ScriptRunner admin page, choose "File" and give the relative path, for example `jira/post_functions/...`, `jira/listeners/...`, `jira/scripted_fields/...`, `jira/behaviours/...`, `jira/jobs/...`.
- **REST endpoints.** Scripts in `src/main/jira/REST_endpoints` use `@BaseScript CustomEndpointDelegate`; add them under ScriptRunner > REST Endpoints. They are then served at `<jira-base-url>/rest/scriptrunner/latest/custom/<endpoint-name>`.
- **Jira Cloud scripts** (`src/main/jira_cloud`) are for ScriptRunner for Jira Cloud, which has no script roots: paste their content into the Cloud console, listener or post-function editor. They compile in section 5.2 only because Groovy resolves their Cloud-specific calls (`get`, `put`, `logger`) dynamically.
- **Confluence scripts** (`src/main/confluence`) are for ScriptRunner for Confluence (console and scheduled jobs).

The scripts target Jira 8.10 / Confluence 7.7 with ScriptRunner 6.9; newer product versions may have changed APIs.

---

## 8. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `Could not transfer artifact ... from/to adaptavist.public (https://nexus.adaptavist.com/content/repositories/external/): nexus.adaptavist.com: Name or service not known` | The repositories in `pom.xml` are dead (2.2). Pass the `settings.xml` from 4.1 with `-s`. |
| `Could not find artifact jta:jta:jar:1.0.1` / `jndi:jndi:jar:1.2.1` / `javax.activation:activation:jar:1.0.2` | Legacy jars not in any public repository (2.3). Extract them from the Atlassian Plugin SDK (4.2). |
| First build seems stuck, hundreds of slow POM downloads for more than 20 minutes | Default dependency collector (2.4). Add `-Daether.dependencyCollector.impl=bf` (Maven 3.9+). Already downloaded files are kept, so simply restart. |
| `Could not transfer artifact ... status code: 502, reason phrase: Bad Gateway` | Transient error from a repository. Run the command again. |
| `No sources to compile` and nothing in `target/` | Expected (2.1). Use section 5.2 to compile the scripts. |
| `unable to resolve class JiraUtilHelper` in `PriorityChange.groovy` and `LinkedIssuesWithCondition.groovy` | Wrong import in the repository (2.6). Exclude both files or fix the import in your copy. |
| `BUG! exception in phase 'semantic analysis' ... Unsupported class file major version 61` (or 65, 69) | Groovy 2.5.13 compiler on JDK 17, 21 or 25. Use JDK 11 for section 5.2. |
| `Error: Could not find or load main class @target.groovyc.args` | JDK 8 does not support `@argfiles`. Use JDK 11. |
| `java.lang.NoSuchMethodError: 'org.apache.logging.log4j.status.StatusLogger ...getLogger()'` | Classpath built with a wildcard (`target/lib/*`). Use the ordered `target/groovyc.args` from 5.2. |
| `The POM for com.onresolve.jira.groovy:groovyrunner:jar:6.9.1 is invalid` and `'dependencies.dependency.version' ... is missing` | Harmless (2.5). |

---

## 9. Quick reference

```bash
# Requirements: JDK 11 + Maven 3.9.x (sections 3 and 4)
# One-time: <SETTINGS_DIR>/settings.xml from 4.1, legacy jars from 4.2

# Resolve all dependencies (prints "No sources to compile", BUILD SUCCESS)
mvn -B -s <SETTINGS_DIR>/settings.xml -Daether.dependencyCollector.impl=bf compile

# Optional: compile-check all scripts (JDK 9-16)
MVN="mvn -B -s <SETTINGS_DIR>/settings.xml -Daether.dependencyCollector.impl=bf"
$MVN dependency:copy-dependencies -DincludeScope=compile -DexcludeArtifactIds=groovy-all -DoutputDirectory=target/lib
$MVN dependency:build-classpath -DincludeScope=compile -DexcludeArtifactIds=groovy-all -Dmdep.prefix=target/lib -Dmdep.outputFile=target/classpath.txt
{ printf -- '-cp '; cat target/classpath.txt; } > target/groovyc.args
find src/main -name '*.groovy' > target/scripts.txt
java @target/groovyc.args org.codehaus.groovy.tools.FileSystemCompiler -d target/scripts-classes @target/scripts.txt
# -> 2 known errors (2.6); exclude those two files to get target/scripts-classes/
```
