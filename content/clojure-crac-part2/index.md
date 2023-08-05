+++
title = "Fast Starting JVM Clojure with Checkpoint/Restore (Part 2)"
date = 2023-08-03

[taxonomies]
tags = ["clojure"]
+++

In [part 1](@/clojure-crac/index.md), we explored how Checkpoint/Restore can be used to improve Clojure startup time without giving up on the power of third-party libraries. In this post, we'll look at ways to actually use third-party libraries with Checkpoint/Restore.

<!-- more -->

## Integrating Clojure CLI with Checkpoint/Restore

When it comes to dependencies in Clojure, there're two popular choices: [Leiningen](https://leiningen.org) and [Clojure CLI](https://clojure.org/reference/deps_and_cli). We'll focus on Clojure CLI in this post, as it is simpler to understand and reverse-engineer.

In short, Clojure CLI is a Bash script that does two things:

1. It looks at your `deps.edn` and downloads necessary dependencies to `~/.m2/repository`
2. It launches Java with the correct classpath

Clojure CLI honors the `JAVA_CMD` and `JAVA_OPTS` environment variables for launching Java. Remember how we created checkpoints in part 1?

```bash
<JDK-CRaC-dir>/bin/java -XX:CRaCCheckpointTo=my_checkpoint -cp clojure-1.8.0.jar clojure.main -e '(jdk.crac.Core/checkpointRestore)'
```

Let's adapt it to utilize Clojure CLI:

```bash
JAVA_CMD=<JDK-CRaC-dir>/bin/java JAVA_OPTS=-XX:CRaCCheckpointTo=my_checkpoint clj -e '(jdk.crac.Core/checkpointRestore)'
```

Clojure CLI will download dependencies and launch Java as usual. The classpath is saved as part of the checkpoint and will be recovered during restore[^1]:

```bash
<JDK-CRaC-dir>/bin/java -XX:CRaCRestoreFrom=my_checkpoint clojure.main
```

There you go! Any libraries that you asked for in `deps.edn` will be available after the restore. Careful though, the classpath is fixed at checkpoint creation, so any modification to `deps.edn` requires recreating the checkpoint.

## Automated checkpoint management

Creating and restoring from checkpoints requires a lot of typing. To make things easier, I created [Clojure-CLI-CRaC](https://github.com/YizhePKU/Clojure-CLI-CRaC), a drop-in Clojure CLI replacement that utilizes Checkpoint/Restore. It detects changes in `deps.edn` and automatically recreate checkpoints when needed. You can find an install guide in the project README. Put it on your PATH and all your Clojure tooling should now launch faster[^2]. Hooray!

## Adding dependencies at runtime

There is an alternative way to use dependencies with Checkpoint/Restore. Thanks to JVM's dynamic nature, classpath can be queried and modified at runtime, which means we can add dependencies after restoring JVM from the checkpoint. [Pomegranate](https://github.com/clj-commons/pomegranate) has been doing it for a while now, and Clojure 1.12 will include support for [adding libraries at runtime](https://clojure.org/news/2023/04/14/clojure-1-12-alpha2#_add_libraries_for_interactive_use). You can use either of these with Checkpoint/Restore.

## Conclusion

By integrating with Clojure CLI, we can make use of arbitrary dependencies while enjoying the speed of Checkpoint/Restore. [Clojure-CLI-CRaC](https://github.com/YizhePKU/Clojure-CLI-CRaC) provides an instant upgrade to Clojure CLI with minimal change to your workflow. Please try it out, and I'm hoping to hear your thoughts and feedback.

## Footnotes

[^1]: We could also use Clojure CLI for the restore, like so:

```
JAVA_CMD=<JDK-CRaC-dir>/bin/java JAVA_OPTS=-XX:CRaCRestoreFrom=my_checkpoint clj
```

However, this is no different than just launching Java manually, since most JVM options are ignored during restore. The classpath is recovered from the checkpoint, not from the command line.

[^2] ...or they could stop working due to [the bug that I mentioned in part 1](@/clojure-crac/index.md#2). That bug causes whitespace in command line arguments to be misinterpreted, so filenames with spaces are likely to be affacted. I've reported the bug to the CRaC team, and hopefully it'll be fixed soon.