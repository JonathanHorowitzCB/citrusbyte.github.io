---
layout: post
title: "Extending Xcode 6 with plugins"
description: "A quick guide of the main topics when creating an Xcode plugin"
date: 2014-10-21 13:22:29
categories: code
author: lucholaf
---

Xcode has a comprehensive SDK that provides everything you need to create beautiful, interactive applications. However, due to its closed-source codebase and no official support for extension hooks it falls short in the realm of social and community supported code. It lacks the flexibility that allows the community to grow and feel at home by adapting and maintaining their own code and tools.

But not everything is bad news: while not official, you can extend Xcode through plugins. Some good info has been written on the topic and how you can boot it up. What I\'ll try to show is how to speed up and clear some misty aspects of the development.

This may look like the Q&A I\'d have liked to have when started.

What can be done with a Xcode plugin?
-------------------------------------

You can make it work like Vim, add colors to the console, improve autocomplete, indent your code, color your variables, [auto import headers](https://github.com/lucholaf/Auto-Importer-for-Xcode) (*cough* self-promotion *cough*) and many many [more](http://nshipster.com/xcode-plugins/) things.

Once your plugin is loaded, you\'re in the same memory space Xcode is, so you could call any method on any instance, even [MethodSwizzling](http://cocoadev.com/MethodSwizzling) in case you want to stand in the way of Xcode calls.


Do I have support from Apple?
-----------------------------

Not really. Although Xcode has a plugin architecture, everything is based on private frameworks, mainly DVKit and IDEKit, so class-dump will be our friend and [expose](https://github.com/luisobo/Xcode-RuntimeHeaders) the symbols we need, so you have to import those headers in your project.


Basic anatomy of a plugin
-------------------------

Xcode has a plugin architecture that searches into `~/Library/Application Support/Developer/Shared/Xcode/Plug-ins` for [bundles](https://developer.apple.com/library/mac/documentation/CoreFoundation/Conceptual/CFBundles/AboutBundles/AboutBundles.html) with an .xcplugin extension.

After loading symbols from the object code, it will call `+ (void)pluginDidLoad:(NSBundle *)plugin` on found classes, so that is your entry door into Xcode.

Let\'s present a common scenario: how to reach the editor component of the IDE. After digging with the dumped headers and with the help of what other developers have found, you could get it like this:

{% highlight Objective-C linenos %}
NSWindowController *currentWindowController = [[NSApp keyWindow] windowController];
if ([currentWindowController isKindOfClass:NSClassFromString(@"IDEWorkspaceWindowController")]) {
    IDEWorkspaceWindowController *workspaceController = (IDEWorkspaceWindowController *)currentWindowController;
    IDEEditorArea *editorArea = [workspaceController editorArea];
    IDEEditorContext *editorContext = [editorArea lastActiveEditorContext];
    IDESourceCodeEditor *editor = [editorContext editor];
}
{% endhighlight %}

Look how `IDESourceCodeEditor` exposes a `NSTextView` instance, a vanilla component you can use when manipulating the source code in the editor.

Generally speaking, you\'ll deal with situations like the one from above, where you need to pick a specific control instance in the IDE in order to handle the control. Another common pattern is to listen for notifications and extract info from them (E.g: the name of the file that was just saved). A good start is to listen to all notifications of which are a lot. It\'s eye-opening to view the log and see the cascade of notifications when a single key is pressed in the editor, you should try it if you haven\'t before.

{% highlight Objective-C linenos %}
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(notificationListener:) name:nil object:nil];
{% endhighlight %}


Where should I start from?
--------------------------

You can use the Xcode 6 [plugin template](https://github.com/kattrali/Xcode-Plugin-Template) and start from there, much better than creating a bundle project yourself.


How to debug and find information when there is no documentation from Apple?
----------------------------------------------------------------------------

The most important tool to develop and debug Xcode plugins is well\.\.\. you know\.\.\. Xcode itself. Yes, although it\'s not that popular on the plugin community, you can debug Xcode from Xcode itself. For a simple guide checkout [Debugging Your Xcode Plugin](http://www.blackdogfoundry.com/blog/debugging-your-xcode-plugin/) by Black Dog Foundry.

Now on Xcode 6 there is an **important** step to avoid crashing: you need to deselect \'Enable user interface debugging\' from the options in the \'Run\' scheme.

For a more rudimentary tutorial checkout [Debugging Xcode Plugins](https://coderwall.com/p/-mgtww) by Delisa Mason.


How should I release my plugin?
-------------------------------

You can make users compile the plugin manually and/or you can also add it to the Xcode plugin manager: [Alcatraz](https://github.com/supermarin/Alcatraz) and its package [repository](https://github.com/supermarin/alcatraz-packages).



Diary of a Xcode plugin developer
---------------------------------

[Common Xcode4 Plugin Techniques](http://www.blackdogfoundry.com/blog/common-xcode4-plugin-techniques/) may be one of the best articles regarding the nuts and bolts of plugins and would serve as a great supplement to this article. But as (almost) always, you\'ll need to find your own path to the goal, here are some tips from what I have found on different aspects of plugin development:

#### Working with projects files

[XcodeEditor](https://github.com/jasperblues/XcodeEditor) has a great API when you have to manipulate project files: inspect headers, list frameworks, add classes, etc. If you need to travel your project files, don\'t forget to take this library with you.

#### Testing

Nothing like the feeling of a good battery of tests that exercise your code while moving internal pieces around. Common usage of `XCTest` with the help of `dispatch_groups` to help asynchronous code testing was all I needed to have a good coverage of the plugin.

#### Background threads

Doing computationally intensive tasks like reading many project files at once could really bring Xcode UX to his knees. So try to make use of `GCD` and `NSOperationQueues` when doing non-UI tasks. Don\'t try to parallelize too much without paying attention, since creating a lot of threads and killing them is not a cheap resource. In my case, I created a NSOperationQueue with controlled concurrency for each project you have in the workspace, that allows to do the indexing of classes/protocols in the background without interrupting the UI thread.

#### Profile

Profile CPU and memory, even the Xcode gauges that give some hint of memory and cpu peaks are useful while debugging. I measured indexing times with `- (NSTimeInterval)timeIntervalSinceDate:(NSDate *)anotherDate;` and then optimize the implementation in order to reduce times. And guess where 80% of the time was spent? I/O. Just checking for file existence was much more time consuming than matching a regexp in a good sized header. I/O can still be a bottleneck even with our fancy SSDs.

#### Use Autorelease Pools

Reduce the peak memory footprint if you need to process lot of info in a single event loop. Using autorelease pools is a good idea while indexing, so creating one per file is a good start.


Conclusion
----------

Go for it and improve Xcode! Objective-C (and maybe Swift) developers are going to thank you so much!