# Tweaking big images

Pharo is special in the sense that the language and IDE are mixed. It means that your application runs together with the IDE.
In some cases, it leads to Pharo images consuming a large amount of memory (aka Big Images). 
The goal of this page is to sum up some tweaking that can be done on Pharo images to improve the performances of such images. 

* [Fast dragging](#fast-dragging)
* [Disable taskbar previews](#disable-taskbar-previews)
* [Tune Garbage Collection](#tune-garbage-collection)
* [Disable Quality Assistant plugin](#disable-quality-assistant-plugin)
* [Disable Spotter plugins](#disable-spotter-plugins)
* [Disable Epicea](#disable-epicea)

## Fast dragging

While dragging windows in Pharo, an animation is run. This animation is not performant and can make the dragging of windows slow in large images.

As a workaround, it is possible to solve this problem by enabling a "fast dragging" option:

```Smalltalk
UITheme currentSettings fastDragging: true
```

> Remark: This option is reset when one change the theme Pharo. Thus, you should execute this code snippet each time you update the theme.

## Disable taskbar previews

While going over the taskbar of Pharo, previews of the images are displayed. This feature can be really slow on big images. 
It is possible to disable it:

```Smalltalk
TaskbarMorph showWindowPreview: false
```

## Tune Garbage Collection 

Pharo is a language using a [garbage collector](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)) for its memory management. 
When objects are created, memory is allocated for them. Once in a while, the memory is cleaned and the garbage collector frees the memory of objects that are not used anymore.

This process can be customized via multiple parameters. To understand the impact of those parameters we explain here how garbage collection works in Pharo.

Because most objects usage stops quickly after their instantiation (we say they have a short life-time), Pharo memory (aka "the heap") is divided into multiple spaces:
- The "Old space" contains objects that survived multiples garbage collections (GC). When it is filled, new memory segments are allocated to the old space.
- The "New Space" contains recently created objects. 

The "New Space" is divided into 3 sub spaces:
- The "Eden" containing the newest objects created (taking 5/7 of the new space)
- Two survivors spaces (1/7 of the new space each) contains the object that survived some garbage collection already

The Garbage Collector execute two kind of cleanings:
- The **scavenge** happens when the Eden is filled at a certain ratio and will clean the Eden. When objects from the eden are still referenced, they are moved to the survivors. There are two kinds of survivors: past survivor space and future survivor space (past space and future space for short). Past and future space swap after each scavenge, which empties eden and past space, copying survivors into future space, tenuring overflow to old space and then making future space the new past space. When the survivor space is fill at more than 90%, the survivors are moved to the old space.
- The **full garbage collect** (aka full GC) happens when the heap grow past a certain ratio. It will go through the old space and clean the old objects.

We can customize multiple parameters.

The size of the new space can be increased. When it increases, scavenges will happen less often.
This can increase a lot the speed of algorithm creating a lot of new objects. 

The size of the eden can be accessed via the vm parameter 44. The desired size of the new space can be set via the vm parameter 45. 

By default, the new space size is 3.8Mo. It can be increased to 32 or 64Mo if you have enough memory.

```Smalltalk
Smalltalk vm parameterAt: 44. "9057248"
Smalltalk vm parameterAt: 45 put: 64 * 1024 * 1024
```

> Since the eden size is fixed at startup, this change requires to restart the image.

If you create at some point a lot of objects that will stay for a while, you can increase the size of new memory segments to allocate in order to spend less time to allocate memory. 

By default the growth is of 16Mo.

```Smalltalk
Smalltalk vm parameterAt: 25 put: 32 * 1024 * 1024
```

If your memory needs to grow and shrink a lot, you can also increase the memory threshold above which shrinking object memory happens. 

By default, 32Mo.

```Smalltalk
Smalltalk vm parameterAt: 24 put: 64 * 1024 * 1024
```

You can also increase the ratio of growth and image size at or above which a GC will be performed	post scavenge.

By default, 0.33. This parameter needs to be a float. 

If you use a really high ratio such as 200.0+, no full GC will happen.

```Smalltalk
Smalltalk vm parameterAt: 55 put: 0.7
```

> Note: Those 3 last parameters are reset during image startup. 
> In order to maintain them, you can use startup actions. See [Session management documentation](SessionsManagement.md).

> This might change in the future (https://github.com/OpenSmalltalk/opensmalltalk-vm/issues/477).

By tweaking those parameters, algorithm that took 9min30 before are now taking less than 5min with images of 1.3Go.

Thanks to [Eliot Miranda](http://www.mirandabanda.org/cogblog/microbio/) and [Clément Bera](https://clementbera.wordpress.com/) for the explanations.

For more information: [https://clementbera.wordpress.com/2017/03/12/tuning-the-pharo-garbage-collector](https://clementbera.wordpress.com/2017/03/12/tuning-the-pharo-garbage-collector)

## Disable Quality Assistant plugin

The code browser in Pharo has a plugin to display code critics. This plugin can take some time to compute the rules in case of large images. It is possible to disable it:

```Smalltalk
ClyCriticBrowserPlugin disable
```

## Disable Spotter plugins

Pharo's spotter has many plugins to search for and display results of mny kind.

Some of those searchs can be long to compute in big images. It is possible to disable the extensions that do not interest us.

You can find the list of extensions executing:

```Smalltalk
GTSpotter spotterExtendingMethods
```

Then you caqn disable an extension. For example if you have poor network you can disable the extension looking for projects in the Pharo catalog.

```Smalltalk
GTSpotterExtensionSettings disabledExtensions add: 'GTSpotter_spotterCatalogProjectsFor'
```

The name of the extension to disable should be of this form: `ClassContainingTheExtension_nameOfTheMethodWithoutLastSemicolon`.

In case of images with a lot of code, it is possible that many projects extend the world menu. In this case the world menu plugin can be long to search. You can diable it this way:

```Smalltalk
GTSpotterExtensionSettings disabledExtensions add: 'GTSpotter_spotterWorldMenuFor'
```

The extensions to disable will most probably depend on what fills the size of your image. Look through the list and estimate what can take timefor you and what do not interest you. 

## Disable Epicea

When you compile code, all the changes are persisted in files by the **Epicea** tool. This allows one to revert/replay changes and to get back lost changes in case an image crash. 

Disabling Epicea can speed up the work in large images:

```Smalltalk
EpMonitor reset
````

> WARNING: Disabling Epicea will remove the possibility to replay changes in case your image crash. Disable it only if necessary!
