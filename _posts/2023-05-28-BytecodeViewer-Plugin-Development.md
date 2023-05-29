---
title: BytecodeViewer Plugin Development 
date: 2023-05-28
categories: [Reversing, Tool Development]
tags: [bytecodeviewer]    
---

# Background

There are several publicly available java compilers (jadx,jd-gui etc.) and one of my favorites is [BytecodeViewer](https://github.com/Konloch/bytecode-viewer/tree/ca894b3d1666a39280918198934ed165274846ae). It is a nice gui tool with various decompilers and plugin support. It is in active development and the author seems to accept pull requests and do maintenance to fix bugs and security issues. I have been actively using it for security research and took some of my free time to learn its api and develop a plugin to accelerate the process.


# Run

The tool is distributed as a jar file and run it with the command `java -jar Bytecodeviwer.jar`:

![BytecodeViewer](/assets/img/bytecodeviewer.png)


# Plugin Set Up

## Java

It is already supported by default and do not have to do additional set up

## Python

```bash
curl https://repo1.maven.org/maven2/org/python/jython-installer/2.7.3/jython-installer-2.7.3.jar --output installer.jar
java -jar installer.jar
# after the installation
java -cp "/path/to/jython.jar:/path/to/Bytecode-Viewer.jar" the.bytecode.club.bytecodeviewer.BytecodeViewer
```

For testing, use the skeleton [example](https://github.com/Konloch/bytecode-viewer/blob/ca894b3d1666a39280918198934ed165274846ae/plugins/python/skeleton.py) provided by the tool:


![BytecodeViewer-Python](/assets/img/bytecodeViewer-python.png) 

## Ruby

```bash
curl https://repo1.maven.org/maven2/org/jruby/jruby-dist/9.4.2.0/jruby-dist-9.4.2.0-bin.zip --output installer.zip
unzip installer.zip -d installer
java -cp "/path/to/jruby.jar:/path/to/Bytecode-Viewer.jar" the.bytecode.club.bytecodeviewer.BytecodeViewer 
```


For testing, use the skeleton [example](https://github.com/Konloch/bytecode-viewer/blob/ca894b3d1666a39280918198934ed165274846ae/plugins/ruby/Skeleton.rb) provided by the tool:

![BytecodeViewer-Ruby](/assets/img/bytecodeViewer-ruby.png) 

## Javascript

It is already supported by default and do not have to do additional set up

## Groovy

```bash
curl https://groovy.jfrog.io/artifactory/dist-release-local/groovy-zips/apache-groovy-binary-4.0.12.zip --output installer.zip
unzip installer.zip -d installer
java -cp "/path/to/groovy/lib/*:/path/to/Bytecode-Viewer.jar" the.bytecode.club.bytecodeviewer.BytecodeViewer 
```

For testing, use the skeleton [example](https://github.com/Konloch/bytecode-viewer/blob/ca894b3d1666a39280918198934ed165274846ae/plugins/groovy/Skeleton.gy) provided by the tool:


![BytecodeViewer-Groovy](/assets/img/bytecodeViewer-groovy.png) 

## Full Installer

```python
#!/usr/bin/env python3

import os
import pexpect
import subprocess
import sys

project_root = "/path/to/project"

print (f"[*]Starting Environment Set Up....")

if not os.path.isdir(project_root):
    os.mkdir(project_root)


os.chdir(project_root)


if not os.path.isdir(f"{project_root}/jython"):
    os.mkdir(f"{project_root}/jython")

if not os.path.isdir(f"{project_root}/jruby"):
    os.mkdir(f"{project_root}/jruby")

if not os.path.isdir(f"{project_root}/groovy"):
    os.mkdir(f"{project_root}/groovy")


print (f"[*]Current Path:{os.getcwd()}")
print (f"[*]Installing Jython....")

p = subprocess.Popen(["curl","https://repo1.maven.org/maven2/org/python/jython-installer/2.7.3/jython-installer-2.7.3.jar","--output","jython-installer.jar"])
p.wait()


r = pexpect.spawn("java -jar jython-installer.jar --console")
r.logfile = sys.stdout.buffer
r.expect(">>>")
r.sendline("e")
r.expect(">>>")
r.sendline("n")
r.expect(">>>")
r.sendline("y")
r.expect(">>>")
r.sendline("2")
r.expect(">>>")
r.sendline("n")
r.expect(">>>")
r.sendline("n")
r.expect(">>>")
r.sendline(f"{project_root}/jython")
r.expect(">>>")
r.sendline("y")
r.expect(">>>",timeout=120)
r.sendline("n")
r.close()

p = subprocess.Popen(["rm","jython-installer.jar"])
p.wait()


print (f"[*]Current Path:{os.getcwd()}")
print (f"[*]Installing Jruby....")

p = subprocess.Popen(["curl","https://repo1.maven.org/maven2/org/jruby/jruby-dist/9.4.2.0/jruby-dist-9.4.2.0-bin.zip","--output","jruby-installer.zip"])
p.wait()


p = subprocess.Popen(["unzip","jruby-installer.zip"])
p.wait()

p = subprocess.Popen(["mv","jruby-9.4.2.0/*","jruby"])
p.wait()

p = subprocess.Popen(["rm","-rf","jruby-9.4.2.0"])
p.wait()

print (f"[*]Current Path:{os.getcwd()}")
print (f"[*]Installing Groovy....")

p = subprocess.Popen(["curl","https://groovy.jfrog.io/artifactory/dist-release-local/groovy-zips/apache-groovy-binary-4.0.12.zip","--output","groovy-installer.zip"])
p.wait()

p = subprocess.Popen(["unzip","groovy-installer.zip"])
p.wait()

p = subprocess.Popen(["mv","groovy-4.0.12/*","groovy"])
p.wait()

p = subprocess.Popen(["rm","-rf","groovy-4.0.12"])
p.wait()

print ("[*]Done!")

```

## Launcher Script

```bash
#!/bin/bash
PROJECT_ROOT="/path/to/project/root"
java -cp "$PROJECT_ROOT/jython/jython.jar:$PROJECT_ROOT/jruby/lib/*:$PROJECT_ROOT/groovy/lib/*:/path/to/Bytecode-Viewer.jar" the.bytecode.club.bytecodeviewer.BytecodeViewer
```

# Basic Plugin Development

After setting up the correct environment, you should choose your language to write a plugin in. Initially, I did write a plugin in java but for the purpose of this blog, I will write in jython. 

As a starter, lets take a look at the source code of bytecodeviwer's `Plugin.java`:

```java
public abstract class Plugin extends Thread
{
    //as long as your code is being called from the execute function
    // this will be the current container
    public ResourceContainer activeContainer = null;
    
    @Override
    public void run()
    {
        BytecodeViewer.updateBusyStatus(true);
        
        try
        {
            if (BytecodeViewer.promptIfNoLoadedResources())
                return;
    
            executeContainer();
        } catch (Exception e) {
            BytecodeViewer.handleException(e);
        } finally {
            finished = true;
            BytecodeViewer.updateBusyStatus(false);
        }
    }

    private boolean finished = false;

    /**
     * When the plugin is finally finished, this will return true
     *
     * @return true if the plugin is finished executing
     */
    public boolean isFinished() {
        return finished;
    }

    /**
     * If for some reason your plugin needs to keep the thread alive, yet will
     * still be considered finished (EZ-Injection), you can call this function
     * and it will set the finished boolean to true.
     */
    public void setFinished() {
        finished = true;
    }
    
    /**
     * On plugin start each resource container is iterated through
     */
    public void executeContainer()
    {
        BytecodeViewer.getResourceContainers().forEach(container -> {
            //set the active container
            activeContainer = container;
            
            //call on the plugin code
            execute(new ArrayList<>(container.resourceClasses.values()));
        });
    }
    
    /**
     * On plugin start each resource container is iterated through,
     * then this is called with the resource container classes
     *
     * @param classNodeList all the loaded classes for easy access.
     */
    public abstract void execute(List<ClassNode> classNodeList);
}
```

Thanks to nice comment, it is straight forward to understand the plugin starting process. As mentioned in the comment, iterated resource container gets passed to `execute()` which is what the plugin dev is required to override. It provides a list of class nodes to do whatever operations and analysis devs want to do. This is great but there is a little thing to consider for jython. The parameter `classNodeList` in jython is actually represented as a plugin class obj so it is necessary to actually call `container.resourcesClasses.values()` directly in order to work on each class node. The equivalent code would be something like this:

```python

from the.bytecode.club.bytecodeviewer.api import Plugin
from the.bytecode.club.bytecodeviewer.api import PluginConsole
from java.lang import System
from java.lang import Boolean
from java.util import ArrayList
from org.objectweb.asm.tree import ClassNode

#
# This is a skeleton template for BCV's Ruby Plugin System
#
# @author [Your Name Goes Here]
#

class skeleton(Plugin):
    def execute(classNodeList, notUsed): #for some reason it requires a second arg
        gui = PluginConsole("Baisc Plugin Test")
        gui.setVisible(Boolean.TRUE)

        gui.appendText("Type:%s" % (type(classNodeList)))
 	for clsnode in classNodeList.activeContainer.resourceClasses.values():
		gui.appendText(clsnode.name)

```

Running this against the example [apk](https://github.com/rewanthtammana/Damn-Vulnerable-Bank) yields the following result:

![BytecodeViewer Plguin Test](/assets/img/bytecodeviewer-plugin-test.png)

At this point, I was able to verify that the basic skeleton code actually works and start looking into `ClassNode` docs for available members and methods. My end goal was to write a plugin that searches and identifies the potential native functions suitable for fuzzing. After reviewing through many documentations and code, I was able to come up with the following result:

![BytecodeViewer Plguin Result](/assets/img/bytecodeviewer-plugin-result.png)

By no mean this is complete and in fact I am in the process of adding more features for automation.

# Conclusion

It was a bit of hassle to set up the initial environment for a plugin development but other than that its api and supports were great enough for me to achieve what I needed. For anyone who wants to see more on what can you do with its api, check out BytecodeViewer's [plugins](https://github.com/Konloch/bytecode-viewer/tree/ca894b3d1666a39280918198934ed165274846ae/plugins) directory on github.   
