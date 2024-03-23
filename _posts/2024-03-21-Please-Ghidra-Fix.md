---
title: Make Ghidra Great Again for IDA Users
date: 2024-03-21
categories: [Reversing, Tool Development]
tags: [Ghidra]    
---

# Background

For the past few months or so, I was actively solving CTF challenges and building tools around handling various esoteric binaries. As someone who was heavy IDA user for years, Ghidra was certainly a breath of fresh air into RE space. It is a feature complete and most importantly open source so you can view the source. However, it also has some quirks that made my life a bit difficult. In this post, I will briefly talk about some of the changes I made in order to make my life easier. 

# Key Bindings

This does not require any explanation. Just re-mapped to IDA hotkeys via `Edit` -> `Tool Options` -> `Key Bindings`

# Highlights

Unlike IDA, if you highlight any variable in the decompiled output, it does not highlight all references of said variable. To fix it, navigate to `Edit` -> `Tool Options` -> `Listing Fields` -> `Cursor Text Highlights` and change `Mouse Button To Activate` to `Left`: 

![highlight](/assets/img/ghidra-ida-users-1.png)


# Auto Analysis

By default, IDA does not ask if you want to automatically apply the analysis of said binary whereas Ghidra always prompts with two dialogs. I think (not sure exactly where) I have read somewhere from the developers as to why they decided/had to do it this way but this was a bit too much. Especially, this becomes a bit problem if I want to use for any small CTF challenges. Ghidra can handle these toy problems perfectly fine with default options. To fix it, I cloned the latest dev branch and set up the dependencies:

```
git clone https://github.com/NationalSecurityAgency/ghidra.git
cd ghidra
gradle -I gradle/support/fetchDependencies.gradle init
``` 

Once all build dependencies are downloaded, I made the following patch:

**Ghidra/Features/Base/src/main/java/ghidra/app/plugin/core/analysis/AutoAnalysisManager.java:1104:**

```java
...

boolean askToAnalyze(PluginTool tool) {
    // This code relies on being called in the swing thread to avoid a race condition
    // where multiple threads check the flag before either thread has a chance to set it.
    Swing.assertSwingThread("Asking to analyze must be on the swing thread!");

    // We only ever want to ask once per session even if they said it is ok to ask again
    if (alreadyAskedThisSession) {
        return false;
    }
    alreadyAskedThisSession = true;

    if (GhidraProgramUtilities.shouldAskToAnalyze(program)) {
        String name = HTMLUtilities.escapeHTML(program.getDomainFile().getName());
        HelpLocation help = new HelpLocation("AutoAnalysisPlugin", "Ask_To_Analyze");

        /* <-- comment this section and return true to always analyze
           int result = OptionDialog.showOptionNoCancelDialog(tool.getToolFrame(), "Analyze?",
           "<html>" + name + " has not been analyzed. Would you like to analyze it now?",
           "Yes", "No", "No (Don't ask again)", OptionDialog.QUESTION_MESSAGE, help);

           if (result == OptionDialog.OPTION_THREE) {
           GhidraProgramUtilities.markProgramNotToAskToAnalyze(program);
           }
           return result == OptionDialog.OPTION_ONE; //Analyze
         */
        return true;
    }
    return false;
}
...
```

**Ghidra/Features/Base/src/main/java/ghidra/app/plugin/core/analysis/AutoAnalysisPlugin.java:178**

```java
...
private void analyzeCallback(Program program, ProgramSelection selection) {
    AutoAnalysisManager analysisMgr = AutoAnalysisManager.getAnalysisManager(program);

    analysisMgr.initializeOptions(); // this allows analyzers to register options with defaults

    /* <-- we do not need another dialog, just comment it
       if (!showOptionsDialog(program)) {
       return;
       }

     */

    analysisMgr.initializeOptions(); // reloads the options in case the user changed them

    // check if this is the first time this program is being analyzed. If so,
    // schedule a callback when it is completed to send a FirstTimeAnalyzedPluginEvent
    boolean isAnalyzed = GhidraProgramUtilities.isAnalyzed(program);
    if (!isAnalyzed) {
        analysisMgr.addListener(new FirstTimeAnalyzedCallback());
    }

    // start analysis to set the flag, but it probably won't do more.  A bit goofy but better
    // than the way it was
    //TODO simplify all this after creating a taskManager per program instead of per tool.
    tool.executeBackgroundCommand(new AnalysisBackgroundCommand(analysisMgr, true), program);

    // if has a selection use it
    // if no selection, use all of memory
    analysisMgr.reAnalyzeAll(selection);
}
...
```

Build the patched version and run it. You should see the analysis get applied automatically without any prompts :)

I think there is a field that you can uncheck under `Edit` -> `Tool Options` -> `Options` -> `Auto Analysis` but this still asks if you want to analyze prompt. By making the patch, you do not have to worry about any prompts....:

![highlight](/assets/img/ghidra-ida-users-2.png)


# Manuals

For IDA, [idaref](https://github.com/nologic/idaref) was my go to plugin for referencing instruction manual. Ghidra supports something similar by default. You just have to make sure the reference manual exists under `Ghidra/Processors/<arch>/data/manuals/`. Once you download it to the right location, you can navigate to `Tools` -> `Processor Manual`. One thing to note is that it expects pdf file with a pre-defined name. Just renaming the lastest volume/version of reference manual will solve this error:


![highlight](/assets/img/ghidra-ida-users-3.png)


# Plugins

This is one of the things that I do not like about Ghidra. You have to re-build the plugins everytime new version of Ghidra comes out. Unless there is a whole re-write of API/internal features, I hope this should not be the case for most of the time. To smooth things out, I wrote a quick script to automate the process:


```bash

GHIDRA_HOME=<Ghidra Home>
HOME=/home/<username>/Desktop

# dragondance
cd $HOME/dragondace 
gradle -PGHIDRA_INSTALL_DIR=$GHIDRA_HOME
cp dist/*.zip $GHIDRA_HOME/Extensions/Ghidra 

# ghidra_sega_ldr
cd $HOME/ghidra_sega_ldr
gradle -PGHIDRA_INSTALL_DIR=$GHIDRA_HOME
cp dist/*.zip $GHIDRA_HOME/Extensions/Ghidra 


# GhidraNes
cd $HOME/GhidraNes
gradle -PGHIDRA_INSTALL_DIR=$GHIDRA_HOME
cp dist/*.zip $GHIDRA_HOME/Extensions/Ghidra 


# Ghidra-RSP
cd $HOME/Ghidra-RSP
gradle -PGHIDRA_INSTALL_DIR=$GHIDRA_HOME
cp dist/*.zip $GHIDRA_HOME/Extensions/Ghidra 

# Add More Plugins below this line
# -----------------------------

```


# Comments

If you notice that your comments are not showing up in the decompiled output, check to make sure the options are turned on for any types of comments you are making. You can navigate to `Edit` -> `Tool Options` -> `Decompiler` -> `Display`:


![highlight](/assets/img/ghidra-ida-users-4.png)


Also, while you are there, if you like to see as much information as possible in the decompiled output, you should definitely turn on `Print nULL for null pointers` option.


# Function Graph and Listing Fields


Similar to comments, if any of the fields in the graph node/listing fields are not showing up, press the little white icon at the top corner of either views and expand their size:

![highlight](/assets/img/ghidra-ida-users-5.png)


# Conclusion

I am going to add more things here if I find anything that makes my journey of using Ghidra difficult.


