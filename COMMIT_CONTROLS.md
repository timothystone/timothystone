# Another Caffeinated Day Commit Controls

## Conventions

    The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
    NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
    "OPTIONAL" in this document are to be interpreted as described in
    [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Background

Coupling Checkstyle with `pre-commit` Git hooks is not necessarily the most documented exercise in Javaland. 

A Google search often references a [Stackoverflow post](https://stackoverflow.com/questions/50741967/add-checkstyle-as-pre-commit-git-hook)
and a 14 year old! [GitHub gist](https://gist.github.com/davetron5000/37350) by user [davetron5000](https://gist.github.com/davetron5000).

Dave's PERL implementation works; with an edit or two as recommended in the post comments, but it MAY be showing its 
age.

An earlier—and unreleased—implementation of this package leveraged Dave's script. This implementation required some 
"brute force" facilitation, especially to reduce configuration friction in a developer workflow: platform scripts for
Wintel and *nix/BSD/macOS had to be written to configure Git's `core.hookspath`, `maven-exec-plugin` configurations and 
accompanying Maven profiles were tested and documented, and libraries had to be bundled for execution of the 
`pre-commit` hook.

All of this would be assembled in a zip file using Maven's Assembly Plugin for distribution and installation. 

It was all good fun.

As a "one-off" implementation for a single developer or a small team, the install could be tolerable. For an enterprise,
especially one seeking less "institutional knowledge," the davetron5000 `pre-commit` hook MAY have been unsustainable.

Enter native Maven classpath features and Husky.

Husky removes the implemention overhead of custom scripts for setting Git hooks as required by davetron5000's script. In
coupling Husky and Maven with the [frontend-maven-plugin](https://github.com/eirslett/frontend-maven-plugin) by 
[Eirik Sletteberg](https://github.com/eirslett) a far simpler package is possible. How? The `maven-exec-plugin` uses the
project's dependency classpath, natively, to support Checkstyle in the `pre-commit` hook.

And one gets [commitlint](https://commitlint.js.org) mostly for free. Mostly.

## Husky

Husky is billed as "Modern, native githooks made easy." Commit Controls aims to pre-package a `commit-msg` and 
`pre-commit` hook to ease Javaland integration of these controls.

Husky requires Node and NPM to run, so this makes for an interesting mix in a Java environment. To faciliate the basics 
of commitlint and Checkstyle in the `commit-msg` and `pre-commit` githooks, developers "install" Commit Controls 
and run a Maven profile, at least once, to "configure" Husky, i.e., set Git's `core.hooksPath` configuration.

### Husky "install"

`hookspath` is a Git configuration providing portability of customized Git hooks. Recall, the `.git/hooks` directory is 
not committed to a project's repository and not portable across workstations. Sure, a team lead or team member could 
"onboard" new members. This "institutional knowledge" is easily lost and can be hard on maintainability.

`mvn -Phusky install` runs the necessary features via NPM to point the Git configuration to Husky's Git hooks in the 
target project.

Each developer MUST run this command once to activate the commitlint `commit-msg` and Checkstyle `pre-commit` hooks, but
the project is otherwise ready for use on `clone`.

#### Developers sometimes need a nudge

READMEs for projects SHOULD include specific notes to run the `husky` profile as shown above. However, engineering leads
and merge performers may occasionally find commits on Merge Requests—_Pull Requests_ in GitHub—that clearly were not 
screened by commitlint or Checkstyle.

Missing commitlint commit messages are easily corrected in the merge request by squashing the commit and editing the 
message. However, engineering leads SHOULD reach out to the committer and assist with performing the task and aligning 
with the project expectations, especially where Checkstyle is in place.

### Included Husky Configuration

This package, as noted, has Husky `commit-msg` and `pre-commit` hooks defined.

This configuration was performed as follows:

```bash
# creates a package.json
npm install husky --save-dev
npm install @commitlint/cli --save-dev
npm install @commitlint/config-conventional --save-dev
# creates .husky (and sets core.hookspath locally)
npx husky install
# creates commit-msg hook
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit "${1}"'
# HEREDOC creating commitlint config
cat << EOF >> commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    scope-enum: [1, 'always', []]
  },
};
EOF
```

## Maven <span style="font-family: monospace">pre-commit</span> for Checkstyle Setup

### 0. Extract the release zip

Run `unzip commit-controls-1.0.4 -d /path/to/project/root` to deposit the files on the target project.

:warning: It is possible to extract the release to the Finder (macOS), Windows Explorer, or Linux File Manager and drag 
the files to the target project. Note, *nix/macOS "dot-files," e.g., `.husky`, are "invisible." It MAY be necessary to 
view "dot-files" to fully copy the files to the target project. "Shift-Option-." will allow macOS users to see these 
files if not already displayed. YMMV on Linux.

### 1. Add project properties

It is a good practice to parameterize project dependency and plugin versions. commit-controls assumes that the target 
project POM follows this practice.

```xml
<project>
  ...
  <properties>
    <!-- other properties MAY be present                              -->
    <frontend-maven-plugin.version>1.12.1</frontend-maven-plugin.version>
    <maven-checkstyle-plugin.version>3.2.0</maven-checkstyle-plugin.version>
    <node.lts.version>v16.17.1</node.lts.version>
    <checkstyle.version>10.3.4</checkstyle.version>
  </properties>
  ...
</project>
```

### 2. Add project build pluginManagement plugins

It is a good practice for leverage Maven inheritence features in all projects. If using a parent POM consider using 
`pluginManagement` to "pin" versions in projects "extending" the parent. If using a project build, `pluginManagement` 
helps project and profile builds too.

```xml
<project>
  ...
  <build>
    <pluginManagement>
      <plugins>
        <!-- other plugins MAY be present                           -->
        <plugin>
          <groupId>com.github.eirslett</groupId>
          <artifactId>frontend-maven-plugin</artifactId>
          <version>${frontend-maven-plugin.version}</version>
          <configuration>
            <installDirectory>target</installDirectory>
          </configuration>
        </plugin>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-checkstyle-plugin</artifactId>
          <version>${maven-checkstyle-plugin.version}</version>
          <dependencies>
            <dependency>
              <groupId>com.puppycrawl.tools</groupId>
              <artifactId>checkstyle</artifactId>
              <version>${checkstyle.version}</version>
            </dependency>
          </dependencies>
          <configuration>
            <!-- the checkstyle plugin can be configured here       -->
          </configuration>
          <executions>
            <execution>
              <id>validate</id>
              <phase>validate</phase>
              <goals>
                <goal>check</goal>
              </goals>
            </execution>
          </executions>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>
  ...
</project>
```

See [frontend-maven-plugin](https://github.com/eirslett/frontend-maven-plugin) documentation for detailed configuration 
options.

### 3. Add dependencyManagement and dependencies

Like `pluginManagement`, `dependencyManagement` provides project level inheritence and is especially effective in parent
POMs.

The use of the dependency of Checkstyle here allows the `pre-commit` hook to load Checkstyle directly from the project 
classpath built by Maven.

```xml
<project>
  ...
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>com.puppycrawl.tools</groupId>
        <artifactId>checkstyle</artifactId>
        <version>${checkstyle.version}</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
  ...
  ...
  <dependencies>
    <dependency>
      <groupId>com.puppycrawl.tools</groupId>
      <artifactId>checkstyle</artifactId>
      <!-- `version` is provided by `dependencyManagement`          -->
    </dependency>
  </dependencies>
  ...
</project>
```

### 4. Add a profile to bootstrap Husky

```xml
<project>
  ...
  <profiles>
    <profile>
      <id>husky</id>
      <build>
        <plugins>
        <!-- this is known as a "profile build"                     -->
        <plugin>
          <!-- `version` is provided by `pluginManagement`          -->
          <groupId>com.github.eirslett</groupId>
          <artifactId>frontend-maven-plugin</artifactId>
          <executions>
            <execution>
              <id>install-node-and-npm</id>
              <goals>
                <goal>install-node-and-npm</goal>
              </goals>
              <configuration>
                <nodeVersion>${node.lts.version}</nodeVersion>
                <!-- npm will be extracted from the node distro     -->
              </configuration>
            </execution>
            <execution>
              <id>npm install</id>
              <goals>
                <goal>npm</goal>
              </goals>
            </execution>
            <execution>
              <id>husky install</id>
              <goals>
                <goal>npm</goal>
              </goals>
              <phase>generate-resources</phase>
              <configuration>
                <!-- a run script is provided in the package.json   -->
                <arguments>run husky:install</arguments>
              </configuration>
            </execution>
          </executions>
        </plugin>
        </plugins>
      </build>
    </profile>
  </profiles>
  ...
</project>
```

### 5. Commit

### 6. Run the husky profile

`mvn -Phusky install`

### 7. Test

Test a commit.

## commitlint

Commitlint enables [semantic versioning](https://semver.org) though release tooling such as the Maven Release Plugin.

Commitlint leverages the [Conventional Commits specification](https://www.conventionalcommits.org/en/v1.0.0/) to 
facilitate major, minor, and patch releases.

Let's look at the basics of the specification and what is expected of developers and tech leads.

**Basically, commit messages that don't meet the specification are rejected until corrected.**

### The Basic and Default Commit Format

<pre>
<a href="#types">&lt;type&gt;</a>(<a href="#scopes">&lt;optional scope&gt;</a>): <a href="#subject">&lt;subject&gt;</a>
empty separator line
<a href="#body">&lt;optional body&gt;</a>
empty separator line
<a href="#footer">&lt;optional footer&gt;</a>
</pre>

#### Types
* API relevant changes
    * `feat` commits add a new feature
    * `fix` commits fix a bug
* `docs` commits are documentation only
* `style` commits SHOULD not affect the meaning (white-space, formatting, missing semi-colons, etc.)
* `refactor` commits code changes that neither fixes a bug (`fix`) nor adds a feature (`feat`)
* `perf` commits improve performance
* `test` commits add missing tests or correcting existing tests
* `build` commits affect the build system or external dependencies
* `ci` commits CI configuration files and scripts
* `chore` commits include changes that don't modify `src` or test file, e.g., `.gitignore` changes
* `revert` commit, well, revert a previous commit. 

**Only** `feat` and `fix` generate a release, minor and patch, respectively. A [`footer`](#footer) will generate a major
release, *only* if containing the defined markers of Conventional Commits, i.e., "BREAKING CHANGE:".

#### Scopes
The `scope` provides additional contextual information.
* Don't use issue identifiers as scopes
* Is an **optional** part of the format
* Allowed Scopes depends on the specific project

See [Defining Scopes](#defining-scopes) for more information about enumerating scopes in a project.

#### Subject
The `subject` contains a succinct description of the change.
* Is a **mandatory** part of the format
* Use the imperative, present tense: "change" not "changed" nor "changes"
* Don't capitalize the first letter
* No dot (.) at the end

#### Body
The `body` SHOULD include the motivation for the change and contrast this with previous behavior.
* Is an **optional** part of the format
* Use the imperative, present tense: "change" not "changed" nor "changes"
* This is the place to mention issue identifiers and their relations

#### Footer
The `footer` SHOULD contain any information about **Breaking Changes** and is also the place to **reference Issues** 
that this commit refers to.
* Is an **optional** part of the format
* **optionally** reference an issue by its id.
* **Breaking Changes** SHOULD start with the word `BREAKING CHANGES:` followed by space or two newlines. The rest of the
commit message is then used for in generating a major release.

## Checkstyle

[checkstyle.org](https://checkstyle.org) notes the following:

> Checkstyle is a development tool to help programmers write Java code that adheres to a coding standard. It automates 
> the process of checking Java code to spare humans of this boring (but important) task. This makes it ideal for 
> projects that want to enforce a coding standard.

### Implementation notes

This packaging uses Checkstyle 10+. This version of checkstyle MUST be executed in a Java 11+ development environment. 
As Java 11 is "the new 8" this MAY be a blocker for the target project.

It is possible for developers to use a Java 11+ JDK and target another runtime. Two possible configurations follow:

**`maven.compiler.release`**

Set the following property in the `pom.xml`:

```xml
<!-- other properties MAY be present -->
<maven.compiler.release>8</maven.compiler.release>
```

**maven-compiler-plugin**

In the project `build:pluginManagement` or `build:plugins`, directly configure the plugin as follows:

```xml
<!-- other plugins MAY be present -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>${maven-compiler-plugin.version}</version>
  <configuration>
    <release>8</release>
  </configuration>
</plugin>
```

## Defining Scopes

By default `commitlint` does not define any scopes, thus scopes can be _ad hoc_ in a project.

To define scopes, simply add scopes to the internal array of the `scope-enum` array, provided empty. An example follows:

```javascript

  'rules': {
    'scope-enum': [1, 'always', ['checkstyle', 'config', 'gitlab', 'husky', 'package']],
  },
```

The list of available commitlint rules are here, [commitlint reference](https://commitlint.js.org/#/reference-rules?id=available-rules).
