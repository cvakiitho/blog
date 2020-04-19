---
layout: post
title: "Jenkins Pipelines Debugging REPL"
categories: [programming]
tags: [ programming, devops, jenkins, groovy]
---

Lately I discovered how to simulate REPL, and at least partially debug real Jenkins pipeline job,
I'm not sure if I missed this in official docs, or if it's not there, but I believe this is really useful
especially with larger pipeline project with enough shared libraries to make your head spin.

Writing pipelines is imo hard for a few reasons. First is, documentation is really not great, you are forced to use step generator, and it's (?) icon quite a bit.
Second is, and a major one, the "compile time" for a simplest pipeline is at least 10s. (And it takes at least 1000 compilations before you have a good program _citation missing_)

```
git add -u; git commit --amend --no-edit
git push origin master -f
```

Click Build
Click Dot to see console output

> You could make it faster by using something like https://marketplace.visualstudio.com/items?itemName=dave-hagedorn.jenkins-runner
>
> But good luck with auth, and with servers you aren't admin.


Third one for me until last week was, you can't really debug them, which together with 2. was a real pain.

Last week I discovered you can use `while`, [input](https://www.jenkins.io/doc/pipeline/steps/pipeline-input-step/), and [Groovy Eval](https://docs.groovy-lang.org/latest/html/api/groovy/util/Eval.html) to simulate "REPL" with something like this:

```
while (true) {
  def cmd = input message: 'What to run:', parameters: [string(defaultValue: '', description: '', name: 'cmd')]
  try {
    print "Running ${cmd}"
    print Eval.xy(this, script, cmd)
  } catch (e) {
    print e
  }
}
```

When you pass this to jenkins pipeline, it acts as a breakpoint in code, and you can pass groovy to the `Input requested` in console output.

I like to use this together with Groovy [dump](https://docs.groovy-lang.org/latest/html/groovy-jdk/java/lang/Object.html#dump()) Object method, to see what is inside `this`, `script` inside jenkins shared libraries mostly. Together with simple Groovy scratch file to visualize larger dumps a bit better:

Pasting `x.binding.dump()` to the input creates large single line string `<groovy.lang.Binding@1f7266a2 variables=[steps:org.jenkinsci.plugins.workflow.cps.DSL@3eea148c, pipelineConfig:[general:[stageStatus:[success:SUCCESS, skipped:SKIPPED, failure:FAILURE, unstable:UNSTABLE, notrun:NOT_RUN, aborted:ABORTED]], maven:[mvnTest:[mvnGoal:clean verify, mvnProfiles:[execute.qunit, debug.test.build], mvnOptions:[-f infra/pom.xml],...`

given to this naive scratch script:

```groovy
String dump = '''<groovy.lang.Binding....>
'''
def level = 1
def result = ''
for (def x in dump) {

    if (x =="[") {
      result += '\n'
      result =  result + (' ' * level)
      level++
    }
    if (x == "]"){
      level--
    }
    result += x

}
print result
```

produces something like this:

```groovy
<groovy.lang.Binding@1f7266a2 variables=
 [steps:org.jenkinsci.plugins.workflow.cps.DSL@3eea148c, pipelineConfig:
  [general:
   [stageStatus:
    [success:SUCCESS, skipped:SKIPPED, failure:FAILURE, unstable:UNSTABLE, notrun:NOT_RUN, aborted:ABORTED]], maven:
   [mvnTest:
    [mvnGoal:clean verify, mvnProfiles:
     [execute.qunit, debug.test.build], mvnOptions:
     [-f infra/pom.xml], mvnSystemProperties:
    [mvnGoal:clean deploy, mvnOptions:
     [-f infra/pom.xml], mvnSystemProperties:
    [mvnGoal:clean verify, mvnOptions:
     [-f infra/pom.xml], mvnProfiles:
     [fiori.eslint.build, analysis.plugin]], mvnVerifyNoFailure:
     ...
```

___

#### Notes

* depending on you setup you can get ENV variable with `x.env.dump()`
* it is possible to call pipeline steps with `x.binding.steps.<stepName>("<step params>")`.

```
11:40:44  Running x.binding.steps.sh('date')
[Pipeline] sh
11:40:44  + date
11:40:44  Mon 03 Aug 2020 11:40:44 AM CEST
```


One problem with Groovy `Eval` class is, that it doesn't expose whole stack, for security reasons, so you can inspect only two variables in the breakpoint input loop, usually `this`, and `script`.

All variables you exposed via eval have [binding](https://docs.groovy-lang.org/latest/html/api/groovy/lang/Script.html) which has [steps](https://javadoc.jenkins.io/plugin/workflow-cps/org/jenkinsci/plugins/workflow/cps/steps/package-summary.html)

It is possible to do a full thread dump on jenkins, and deserialize it's state via `<jobName>/lastBuild/threadDump/` and `lastBuild/threadDump/program.xml`, but the enormous xml is hard to navigate, and read. It's ok for searching.
#### Sources:

* [https://notes.asaleh.net/posts/debugging-jenkins-pipeline/](https://notes.asaleh.net/posts/debugging-jenkins-pipeline/)
