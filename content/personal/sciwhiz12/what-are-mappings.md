---
title: "What are Mappings: An Explainer"
date: 2024-11-29T00:33:12+08:00
categories:
- Personal Blog
author: sciwhiz12
summary: "We take at look at one of Minecraft's obstacles for modding: obfuscated names, and the innovation that is *mappings*: a set of names laid on top of the obfuscated ones, which make modding Minecraft more bearable for developers of mods and tools alike."
description: "In this post, we take a look at the topic of mappings: a set of class, method, and field names. We discuss the nature of obfuscated names and the problems they cause, and introduce the concepts of intermediate mappings and human-readable mappings and how they fix the problems of obfuscated mappings. We also briefly discuss how mappings are present in the development and production environments of Minecraft, and interspersed throughout the post are examples of the various mappings in play throughout the Minecraft modding community, including Mojmaps."
---

{{< box note >}}
This article was originally posted by me to the now-defunct Minecraft Forge blog, in July 2022. As such, the information here may be outdated or inaccurate, with respect to current practice as of publication of this repost.

For posterity, this article remains unchanged from its final form. If any updates are made, it will probably be in a new post, with a link to that attached here.
{{< /box >}}

Modding Minecraft has had many challenges since its start in the early 2010s. From the closed-source nature of the game, the lack of user-friendly tools to modify the game code, the imperfect nature of Java decompilers' output, and the hurdles of distributing modified class files, there has certainly been a lot of difficulties in creating and publishing mods for Minecraft, as well as the tools to make these mods.

One of these hurdles is the obfuscated nature of Minecraft. Mojang obfuscates the code of Minecraft for (to my knowledge) two main reasons: to shrink the size of the final jar files[^shortnames], and to protect Mojang's intellectual property. Because of this obfuscation, developing mods for Minecraft in the early days used to be more difficult due to the nature of how the obfuscation is implemented.

Today, we now have **mappings** which define sets of human-readable names (and even some documentation) for classes, fields, and methods. These mappings are applied in the environments of mod developers, bettering their development experience with understandable names.

## Obfuscated Pain

Minecraft uses the [ProGuard code obfuscator and optimizer][proguard] to obfuscate the names of classes, fields, and methods (collectively termed as "member names" in this post) within the published client and server jars. These names are programmatically generated and very short, usually one to three characters in length: `aa`, `dqw`, `xw$c$a`[^dollarsign] to name a few examples.

These obfuscated names, while great for shrinking the size of Minecraft's jar files, are horrible for mod development for a few reasons:

1. Because these obfuscated names are generated by a program automatically, they communicate *nothing* about the actual purpose and logic of the class to the mod developer.

   For example, while a block may be represented by the class named `ciw`, its position may be represented by the class named `gt` while the world it resides in may be represented by `cga`, `etm`, or `afo`[^obfuscated]. Nothing about those names communicate their actual intent, which is highly confusing.

2. As these names are generated for each published version, obfuscated names usually change between each and every new version. This is highly disorienting when porting a mod from the previous version to the next, as the names may now be linked to wildly different classes between those versions.

   For example, the class representing the world where blocks and entities reside in may be named `bqb`, `bwp`, `cad`, and `cga` across multiple versions[^versions].

3. In Java, multiple methods may share the same name, as long as their signatures (basically the method's parameters) are different -- a feature called [method (or function) overloading][overloading]. For example, one can define the methods `int add(int a, int b)` and `long add(long a, long b)` in the same class, and you can call either of them as long as you have the right parameters[^overloadcasting].

   The obfuscator takes advantage of this by naming multiple, totally different methods within a class with the same obfuscated name, as long as their signatures differentiate them enough for Java to know which to call in which case. For example, the methods `close()`, `fill(int[] array)`, `contains(long[] array, long value)` may all be given the same name `a` by the obfuscator, and Java is totally fine with it.

4. In addition to the above point, the Java Virtual Machine is totally fine with methods sharing the same name and same parameters but different return types.

   This is because the Java programming language has no way to specify when invoking a method to "please invoke the one with the return type of `SomeClass`", the JVM bytecode contains the full descriptor for the method being invoked, including the return type.

   Because of this, decompiling the obfuscated code without preprocessing leads to many compile errors due to methods sharing the same name and parameters, as the Java programming language disallows that.

```goat
     .----.                                         .----.         
    | 1.16 +---.            .---------------.      | 1.16 +-.      
     '-+--'    |\          |                |       '-+--'  |\     
       | Level '-o-------->|                *-------->| bqb '-'    
       |         |         |                |         |       |    
       '---------'         |                |         '-------'    
 - - - - - - - - - - - - - |                |- - - - - - - - - - - 
     .----.                |                |       .----.         
    | 1.17 +---.           |                |      | 1.17 +-.      
     '-+--'    |\          |                |       '-+--'  |\     
       | Level '-o-------->|                *-------->| bwp '-'    
       |         |         |                |         |       |    
       '---------'         |                |         '-------'    
 - - - - - - - - - - - - - |                |- - - - - - - - - - - 
     .----.                |                |       .----.         
    | 1.18 +---.           |   Obfuscator   |      | 1.18 +-.      
     '-+--'    |\          |   (ProGuard)   |       '-+--'  |\     
       | Level '-o-------->|                *-------->| cad '-'    
       |         |         |                |         |       |    
       '---------'         |                |         '-------'    
 - - - - - - - - - - - - - |                |- - - - - - - - - - - 
   .-------------.         |                |    .----------.      
  | int call(int) +------->|                *-->| int a(int) |     
   '-------------'         |                |    '----------'      
   .----------------.      |                |    .-------------.   
  | int call(double) +---->|                *-->| int a(double) |  
   '----------------'      |                |    '-------------'   
   .-----------------.     |                |    .--------------.  
  | UUID find(Thread) +--->|                *-->| UUID a(Thread) | 
   '-----------------'     |              o |    '--------------'  
                           '---------------'                       
```

As you can see, these drawbacks can impose difficulties on developers of mods and tools alike. That's why the developers of modding tools made an innovation which is the main topic of our post: *mappings*.

## It's Mappings All The Way Down

To reiterate, **mappings** is broadly defined to be a set of names for classes, methods, and fields. For example, we usually refer to the names generated by the obfuscator for Minecraft as the *obfuscated mappings*.

What the developers of the past did was expand the concept to add two more *layers* of mappings on top of the obfuscated mappings: what we'll call the *intermediate mappings* and the *human-readable mappings*.

### The Middle Man-Mappings

First, let's summarize the problems we have with obfuscated names:

1. They communicate nothing about what they're naming.
2. They change wildly between versions.
3. Multiple methods may have the same name, as long as their return types differ.

We'll focus on the last two problems for now; the first problem will be solved later with human-readable mappings.

To solve these two problems, tool developers came up with the concept of **intermediate mappings**. These mappings map every obfuscated name with a generated but *unique* name, usually implemented by incrementing a counter.

Forge's intermediate mappings are provided through the [MCPConfig project][mcpconfig], giving us the *SRG[^srgmeaning] intermediate mappings*. In SRG mappings, names are in the form of `<type>_####_`[^tsrg2], where `<type>` is a one-character identifier: `C` for classes[^srgclasses], `m` for methods, and `f` for fields.

Other notable intermediate mappings is Fabric's [Intermediary][intermediary] (whose name is apt yet unfortunately confusable with our concept of intermediate mappings) and Quilt's Hashed Mojmap (generated by their [mappings-hasher][mappings-hasher] tool).

This fixes our third problem: because each method has a unique name, there should be no compile errors due to methods sharing the same name. But the second problem is still present; these names will still change across versions, as we've just shifted the problem from changing obfuscated names (`a` becoming `c`) to changing intermediate names (in the case of SRG, `m_1_` becoming `m_3_`).

To solve that, we introduce a new step into the generation of these intermediate names: *carrying over intermediate names* from the previous version for unchanged classes, methods, and fields. This means that we only generate new intermediate names when we cannot figure out if a class, method, or field was kept unchanged from the previous version[^newnames].

The process of carrying over intermediate names is called **matching**, since it involves matching intermediate names from previous versions to the classes, methods, and fields of new versions where possible and appropriate. MCPConfig uses the [JarAwareMapping tool or JAMMER][jammer] (and previously [DePigifier][depigifer]), while Fabric uses the [Matcher][matcher] tool[^quiltmatcher].

```goat
                            .----.    .----.    .----.                              
                            |    |\   |    |\   |    |\                             
                            |  A '-'  |  C '-'  |  D '-'                            
                            |      |  |      |  |      |                            
                            '--o---'  '--o---'  '--o---'                            
.----------------------.        '--------+--------'      .----------------------.   
| IntMap v1            |\                |               | IntMap v2            |\  
| .---.   .---.   .---.| \               v               | .---.   .---.   .---.| \ 
| | A |   | B |   | C |'--'        .-------------.       | | A |   | C |   | D |'--'
| '-+-'   '-+-'   '-+-'   |       | Intermediate |       | '-+-'   '-+-'   '-+-'   |
|   |       |       |     +------>|  Mappings    *------>|   |       |       |     |
|   v       v       v     |       |  Matcher   o |       |   v       v       v     |
| .---.   .---.   .---.   |       '-------------'        | .---.   .---.   .---.   |
|| c_1 | | c_2 | | c_3 |  |   Class A and C get their    || c_1 | | c_3 | | c_4 |  |
| '---'   '---'   '---'   |   existing IDs while         | '---'   '---'   '---'   |
'-------------------------'   class D gets a new ID.     '-------------------------'
                                                                                    
```

For example, let's say the class representing the world where blocks and entities reside in was named `bwp`, in 1.17, and named `cad` in 1.18. Even though the obufscated name is different across both versions, the SRG name for both is still `C_1596_`, as these classes were unchanged enough between each version that we recognize them as being the same class, conceptually.

Now, thanks to intermediate mappings, we've solved two of our problems:

1. Because intermediate names are carried over between versions, these changes don't change wildly between versions; we have a (mostly) stable set of names to build against.
2. Because we give each class, method, and field a unique name, we can never have conflicts between methods (due to having the same name and parameters).

For these reasons, the modded Minecraft world compiles[^reobfuscation] all mods using intermediate mappings. The modded client and server which users play on run using intermediate mappings, which is why you see many `m_####_` and `f_####_` names in crash reports.

### Readable Names FTW

Now we're left with the first problem from our list: these names, both obfuscated and intermediate, communicate nothing about the member they name. Following our previous examples, while `cad` and `C_1596_` refer to the same class, we don't *know* what that class does unless we look into it and investigate it. Its name imparts no meaning information on what is it or what it does.

For this, we introduce the concept of **human-readable mappings**. As its name suggests, this is a set of names that are actually comprehendable by humans and informs developers from its name alone what it is and what it does.

For Minecraft versions 1.14 and above, Mojang publishes the set of human-readable names they use in their own codebase. They do this by publishing the obfuscation logs from ProGuard, which writes out each original source name and its corresponding obfuscated name[^obflogs]. We call these mappings the **Mojang mappings**, often shortened to simply **Mojmaps**.

For Minecraft versions 1.16 and below, there was a community-sourced mappings set, where developers would suggest names (and optionally, javadocs) to a bot which then compiled these into a daily export. Because the bot was named MCPBot (named after the Mod Coder Pack or MCP), these community-sourced mappings are commonly referred to as **MCP mappings**[^mcpmappingsname]. The bot was decommissioned and the creation of new MCP mappings exports was halted shortly after Forge switched to using Mojang mappings. For our purposes, we will ignore MCP mappings and focus on Mojang mappings.

Other notable examples of human-readable mappings include Fabric's [Yarn][yarn] and Quilt's [Quilt Mappings][quilt-mappings].

As an example of Mojang mappings in action, we go back to our trusty example. The class representing the world where blocks and entities reside in would be named `cad` in obfuscated mappings and `C_1596_` in SRG mappings; neither of the names impart the purpose of the class to the reader. In Mojang mappings, the class is named `Level`, which does communicate what the class is. Another example: an obfuscated name of `gh` and SRG name of `C_4675_` has the corresponding Mojang name of `BlockPos`, which tells us that it is a class that represents the position of a block.

### The Mojmaps Exception

Human-readable mappings are built against an intermediate mapping, to preserve names across versions when the human-readable mapping is updated. For example, the MCP mappings are built against the SRG mappings.

The notable exception to the above is Mojang mappings, as they are published in the form of obfuscated names to human-readable names. We regard them as human-readable mappings, but because they are not built against intermediate mappings, they don't neatly fit within our definition of human-readable mappings.

Nevertheless, we will primarily consider Mojang mappings as human-readable mappings. Tools often compensate for Mojang mappings by first remapping it to use the intermediate mappings of choice, then applying that combined mapping to the local environment.

However, it should be noted that Mojang mappings can also perform the role of intermediate mappings, because their nature as the source names used by Mojang means they also solve both problems as solved by intermediate mappings -- cross-version stability and method name non-collision. In practice, there is no known modloader as of writing that uses Mojang mappings as their intermediary mappings.

## Bringing It All Together

Now we have the concepts of intermediate mappings and human-readable mappings, which solve all three of our problems combined: intermediate names provide version-stable and non-colliding names, while human-readable names provide understandable names for the benefit of developers.

Modding tools and their developers conceptualize the three types of mappings -- obfuscated, intermediate, and human-readable mappings -- into three **mapping layers** across the development environment (where mod developers make mods) and the production environment (where users play with mods).

Each type of mappings is overlaid on top of each other to provide the mappings used in both environments.

- For the production environment, we simply overlay the intermediate names on top of the obfuscated names, so it works in *intermediate names*. Users don't use the human-readable mappings as they're usually not visible to them (except for crash reports or logging files).
- For the development environment, we take the intermediate names from the production environment and overlay the human-readable names on top, so it works in *human-readable names*. This means the developers can work with names they can understand and comfortably work with, both in their IDE and when reading their logging files for debugging purposes.

```goat
       .------------------------------------------------------------------.     --.  --.               
       |                         Obfuscated Names                         |        |    |              
       '-------------+------------------+-----------------+----------+----'        |    |              
                     |                  |                 |          |             |    +-----------.  
 .------------.      |                  |                 |          |             |   | Production  | 
| Intermediate | --- | --- --- --- ---  | --- --- --- --- | ---  --- |-.           |   | environment | 
|   Mappings   |     |                  |                 |          |  |          |    +-----------'  
 '------------'      v                  v                 v          |             |    |
         | .------------------. .--------------. .----------------.  |  |          |    | (includes    
           |                  | |              | |                |  |             |    |  obfuscated &
         | | MCPConfig or SRG | | Intermediary | | Hashed Mojmap  |  |  |          |    |  intermediate
           |                  | |              | |                |  |             |    |  layers)     
         | '---------+--------' '-------+------' '--------+-------'  v  |          |    |              
                     |                  |                 |        .-----------.   |    |              
         |           |                  |                 |        |           |   |    |              
          '- --- --- | --- --- --- ---  + --- --- --- --- + --- ---|           |   |    |              
                     |                  |                 |        |  Mojang   |   |    |              
 .--------------.    +                  +                 +        | (Mojmaps) |   | --'               
| Human-readable |-- |\--- --- --- ---  |\--- --- --- --- |\--- ---|           |   |                   
|    Mappings    |   | '----------------)-+---------------)-+----->|           |   +-----------.       
 '--------------'    v                  v                 v        '-----------'  | Development |      
           .------------------. .--------------. .----------------.               | environment |      
         | |                  | |              | |                |     |          +-----------'       
           |  MCP or MCPBot   | |     Yarn     | | Quilt Mappings |                | (includes         
         | |                  | |              | |                |     |          |  obfuscated,      
           '------------------' '--------------' '----------------'                |  intermediate, &  
         |                                                              |          |  human-readable   
          '- --- --- --- --- --- --- --- --- --- --- --- --- --- --- --'           |  layers)          
                                                                               ---'                   
```

The production environment's intermediate mappings are installed onto your Minecraft instance when you install a modloader like Minecraft Forge or Fabric. They take the original Minecraft jar from your computer and remap it in place[^inplace] from its original obfuscated mappings to the intermediate mappings of the modloader's choice.

The development environment's human-readable mappings are handled by your modloader's tool (which is commonly a Gradle plugin nowadays): Forge's [ForgeGradle][forgegradle], Fabric's [fabric-loom][fabric-loom], Quilt's [quilt-loom][quilt-loom], Spigot's [BuildTools][buildtools], Sponge's [VanillaGradle][vanillagradle], and others.

### Mods Across Mappings

As seen above, the two environments use different mappings, which complicates matters when compiling and distributing your mod. Because your development environment uses human-readable mappings, trying to use the raw jar from compilation without any processing leads to errors due to the mismatch in names.

Fortunately, your modloader's tool of choice can remap your mod from using your development environment's human-readable mappings to the intermediate mappings expected in the production environment. This process is called **reobfuscation**[^obfuscationconfusion].

A similar situation arises when trying to load a reobfuscated mod in intermediate mappings in your development environment, which uses human-readable mappings. For this, your modloader's tool of choice can remap the mod in the same but opposite way as described above, in a process called **deobfuscation**[^obfuscationconfusion].

## Conclusion

Now you understand the vital role that mappings -- both intermediate and human-readable -- have in the modded Minecraft scene. We've come a long way since the beginning of the Minecraft modding, with innovations such as the move from the Mod Coder Pack's Python and batch scripts to the various modloaders' Gradle plugins for their own environments, and the birth of new and exciting modloaders and their own mappings sets.

The publication of Mojang mappings since 1.14 (and the eventual license put down by Mojang and accepted by the community) went a long way in unifying the mappings and environments of various modloaders. Today, Mojang mappings acts as the *lingua franca* between developers of each community, uniting developers with common class, method, and field names while helping each other, as Mojang intended[^dinnerbonetweet].

[^shortnames]: Since all members names are stored in the defining class as well as any class which references the members of that class, making them as short as possible (as done by the obfuscator) saves some space, which adds up when taking everything into consideration. Furthermore, all packages are flattened to the root package, so it reduces the length of class names from e.g. `net/minecraft/world/level/Level` to `cga`.
[^dollarsign]: The dollar sign character is used by Java to indicate nested classes within classes. In this case, `xw$c$a` means the outermost class `xw`, which has a nested class `c`, which itself has a nested class `a`.
[^obfuscated]: These names are taken from Minecraft 1.19. For the curious, their actual Mojang names are `Block`, `BlockPos`, `Level`, `ClientLevel`, and `ServerLevel` respectively.
[^versions]: The versions are Minecraft 1.16, 1.17, 1.18, and 1.19 respectively.
[^overloadcasting]: As a small trivia note: if a developer wishes to, for example, call the `long` version of `add` even if they have `int` variables to pass in, they can merely cast the arguments to `long`s and Java will know that they intend to use the `long` overload: `long sum = add((long) a, (long) b)`.
[^srgmeaning]: SRG stands for "**S**earge's **R**etro**G**uard", where "Searge" refers to [SeargeDP][seargedp], the founder of the Mod Coder Pack and "RetroGuard" refers to a tool which aimed to partially reverse (or one could say, retrograde) the obfuscation applied by ProGuard.
[^tsrg2]: Between 1.16 and 1.17, MCPConfig moved from the original TSRG format to the new TSRGv2 format and the new format for SRG names. In the original SRG naming format, classes were `C_####_` (though these only existed during the matching process), methods were `func_####_obf_`, fields were either `field_####_obf_`, and parameters were either `p_(method ####)_#_` or `p_i(method ####)_#_` for constructor parameters. Older SRG names did not have the final underscore.
[^srgclasses]: Developers will normally never encounter SRG names for classes like `c_231_`, as the Forge toolchain automatically replaces SRG class names with Mojang class names, even in the production environment.
[^newnames]: This happens in chiefly two cases: when a member was modified to the point of being unrecognizable compared to its previous version's counterpart, or a member was newly added. Simplifying it down, we essentially do what Git does to detect renames: try to match to a previous file within a threshold, or just declare it a newly added file.
[^quiltmatcher]: Quilt does not follow the traditional method of matching previous versions of their intermediary mappings to create new versions. Instead, Hashed Mojmap is, as its name implies, the Mojmap name but hashed according to [a defined specification][hashedmojmapspec]. The driving idea we will note for our purposes is that a change of name of a member usually signals a large enough change that it warrants a new intermediate name. Conversely, the lack of a change in the name usually signals that the changes are minor enough that a new intermediate name is not needed.
[^reobfuscation]: To be more accurate, the compiled mods are put through a processor which changes the names they use from human-readable mappings to intermediate mappings, in a process commonly called *reobfuscation*. Note that this shouldn't be confused with changing names to *obfuscated* names -- that's simply the process of obfuscation.
[^obflogs]: These log files are generated to allow users of ProGuard to deobfuscate stack traces generated by obfuscated versions of their programs, from nonsensical obfuscated names to the source names the program's developers are familiar with. ProGuard calls these obfuscation logs as "mapping files" (which we don't use in this post to avoid confusion), and the tool for deobufscating stack traces is called [ReTrace][retrace].
[^mcpmappingsname]: Though it is commonly referred to as MCP mappings by the community, LexManos, the lead developer of Forge, has said in the past that the name "MCP mappings" is incorrect as the MCP toolset is obsolete, preferring the terms "crowdsourced mappings" or "bot mappings". According to Curle, the community manager for Forge, the technical name for the mappings is "MCPBot Export".
[^inplace]: This remapping of the Minecraft jar is done at the computer of the user, as the [Minecraft End User License Agreement][mc-eula] forbids redistributing modified versions ("Modded Versions") of Minecraft. In fact, all modloaders' environments are cleverly built to prevent breaking the EULA by distributing the changes needed to apply to the base game to get a modded game, instead of distributing the modded game outright.
[^obfuscationconfusion]: Not to be confused with the process of **obfuscation**, which remaps something to obfuscated names, and is often performed by an obfuscator. The origin of the term "reobfuscation" (and the related term "deobfuscation") in this context likely heralds back to the era of the Mod Coder Pack (then named the Minecraft Coder Pack), where it called mapping the Minecraft class files from obfuscated names to intermediate names as "deobfuscation" and the reverse process as "reobfuscation".
[^dinnerbonetweet]: {{<tweet user="Dinnerbone" id="1293597329854009344">}}

[proguard]: https://www.guardsquare.com/proguard
[overloading]: https://en.wikipedia.org/wiki/Function_overloading
[mcpconfig]: https://github.com/MinecraftForge/MCPConfig
[intermediary]: https://github.com/FabricMC/intermediary
[mappings-hasher]: https://github.com/QuiltMC/mappings-hasher
[jammer]: https://github.com/OrionDevelopment/JarAwareMapping
[depigifer]: https://github.com/MinecraftForge/DePigifier
[matcher]: https://github.com/FabricMC/matcher
[seargedp]: https://twitter.com/SeargeDP
[hashedmojmapspec]: https://github.com/QuiltMC/rfcs/blob/master/rfc/0019-hashed-mojmap.md
[retrace]: https://www.guardsquare.com/manual/tools/retrace
[yarn]: https://github.com/FabricMC/yarn
[quilt-mappings]: https://github.com/QuiltMC/quilt-mappings
[forgegradle]: https://github.com/MinecraftForge/ForgeGradle
[fabric-loom]: https://github.com/FabricMC/fabric-loom
[quilt-loom]: https://github.com/QuiltMC/quilt-loom
[buildtools]: https://hub.spigotmc.org/stash/projects/SPIGOT/repos/buildtools
[vanillagradle]: https://github.com/SpongePowered/VanillaGradle
[mc-eula]: https://www.minecraft.net/en-us/eula
