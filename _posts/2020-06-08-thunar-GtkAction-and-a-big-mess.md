---
layout: post
title:  "thunar, GtkAction and a big mess"
date:   2020-06-04 22:51:00
tags: intro
comments: true
---

My journey into the GtkAction abysses of [thunar](https://gitlab.xfce.org/xfce/thunar) began in the mid of 2019. Be warned, it is no story of success. It is rather a story about permanent failure, about finding a way through a mace while walking into almost every dead end.

Actually I just wanted to fix [#198 (Merge all file-context-menus into one)](https://gitlab.xfce.org/xfce/thunar/-/issues/198). But than things got weird. More than half a year later and after numerous interactive rebases I finally merged my branch into master \o/

.. but lets start at the beginning:

The old Thunar used to create the same menu items in different places using different code. In the past that led to inconsistencies. E.g. the location bar only provided a very minimal context menu, no custom actions at all.

<add picture>

From time to time I found myself right-clicking on a location-button, just to find out that there still is no custom action. At some point of maximal annoyance I decided to fix that problem ... not sure if I would have done so when I knew how long that road would be.

Looking at thunar-location-buttons.c revealed a lot of duplicated code. The thunar-standard-view and thunar-window both used the deprecated GtkActionEntry to define menu item labels and related actions. The location buttons just mirrored parts of that code. On top some other actions were defined in thunar-launcher or had their own classes, inheriting GtkAction.

So yay, lets just copy+paste the missing stuff to the location buttons?
Nah, that would be too easy. As a developer who values [DRY](https://de.wikipedia.org/wiki/Don%E2%80%99t_repeat_yourself), it would hurt my belief in clean code to produce more mess.

I started to do some coding .. first I created a new widget "thunar-menu" which internally is a "gtk-menu", and moved menu-item creation and the related actions for copy/cut/paste/delete/move_to_trash to there to have them at some central place, to be reused by different menus. I as well moved the action from thunar-launcher to thunar-menu (I guess the original intention of the launcher was, to actually launch thing, not to manage menu-items) And I replaced separate Actioning classes in favour of methods inside thunar-menu.

Meanwhile the location-button-menu and the context-menu, which I used for testing, were populated with some items again.

The old code made massive use of the deprecated GtkAction and GtkActionEntry classes together with GtkUiManager. I did not want to add more G_GNUC_BEGIN_IGNORE_DEPRECATIONS, to silence warnings. So I decided to replace the deprecated calls.
Looking into the gtk3 documentation revealed that there is "GAction" and "GActionEntry" which provides some service around accelerator activation, and there is GtkMenu/GMenu for which at that time I had no clear idea why there are two of them.

The doc. of GAction told that it should not be used for anything related to labels, icons or creation of visual widgets .. damn. At that time I did not see an advantage in using that class. I decided to rather go for GtkMenu together with some custom replacement for GtkActionEntry: XfceGtkActionEntry.

Retrospective ignoring "GAction" might not have been my smartest move. Meanwhile I understood how GAction can be used with GtkMenu, and I will most likely go for that combination at some later point.

Regarding GtkUiManager: The definition of menu-items of thunar were scattered across 7 different *-ui.xml files, making it hard to figure out what belongs together. Because of that I decided to just get rid of GtkUiManager, and create menu-items in the code instead of predefining their order in xml. IMO the usage of xml files to build GUI's might be nice for static GUI's, though IMO for dynamic menu-creation it just introduces unnecessary complexity.

So I started to build XfceGtkActionEntry and some support methods. XfceGtkActionEntry is a structure which holds labels, tooltips, icons, types, the accelerator paths and a callbacks to the related actions. Since it is just a struct, it can be filled in a static way, just like GtkActionEntry.

Next problem: The menus in thunar so far did not get destroyed, but were updated whenever the selected items got changed, and got shown when needed. That sounded wrong to me. Why should I want to update menu-items while no menu is visible at all ?
There were bugs about menu-flickering and slowness while rubber banding/mass select which seem to be related. Since I anyhow needed to touch that part, I decided to build menus only when they need to be shown.

Things went well, I came to the point where I needed some items from thunar-window, like the zoom-section and the view-specific settings. As well some other items from thunar-window did not work any more, since I moved file-menu functionality from thunar-launcher to thunar-menu. So next step clearly was: Introduce XfceGtkActionEntry to the thunar-window menu .. and than shit hit the fan.
So far the window-menu was always present and took care for any accelerator actions. Since my concept was "create menu on request", there was no menu-instance which could take care for accelerators any more, leading to dysfunctional accelerator keys, rendering my whole concept as faulty .. aargh.

After some time of grieve and doubts I fixed the problem by moving most of the code from thunar-menu back to thunar-launcher, which lifetime is coupled to thunar-window.
From now on thunar-menu was more or less just an convenience wrapper for thunar-launcher ... still useful, but sadly it lost its glory. Thunar-launcher now builds volatile menu items on request and permanently listens to the related accelerators. Finally accelerators started to work, and I was able to continue to fight with the window-menu.
I had much more trouble with that menu, too much to tell it here .. however somehow I managed to get it functional, so that it mostly worked like before.

Later on I found out that the class gtk_accel_map, which I use a lot is going to be deprecated soon. So it seems like I will need to touch the accelerator part again. This time I plan to make use of the GActionMap interface .. going to be a story for another day.

For first testing and code-review I luckily I got support of some early adopters. They found many more defects and regressions which kept me busy a long while. Though luckily nothing concept-breaking.

While writing this, there are still some regressions which I introduced, waiting to get fixed by me:
* [Regression: Window menu not updated when using ALT+mnemonic key](https://gitlab.xfce.org/xfce/thunar/-/issues/320)
* [Regression: Window menu not shown when using ALT+mnemonic key directly after starting thunar](https://gitlab.xfce.org/xfce/thunar/-/issues/321)
* [GObject-WARNING on closing thunar in some conditions](https://gitlab.xfce.org/xfce/thunar/-/issues/319)

And there are some tasks on my agenda, for which I just did not find the time so far:
* rename thunar-launcher to thunar-action-manager
* use thunar-menu in bookmark view
* use thunar-menu in Tree view
* and many minor things

Finally I ended up with 25 commits and +4717 / -7149 line changes. The occurrence of G_GNUC_BEGIN_IGNORE_DEPRECATIONS got reduced from 250 to 35, which will further drop when using GtkMenu for bookmark-view/tree-view. That should simplify the move to gtk4 in the future. So overall, the result does not look too bad I guess.

I hope you enjoyed that journey into the thunar internals ... enough storytelling for now, I really need to take care of these remaining regressions !

Many thanks to Reuben, AndreLDM, DarkTrick and others for early testing !
And as well thanks to AndreLDM for permitting me to use the code of [his blog](https://andreldm.com){:target="_blank"} as a base for this very first blogpost of mine!
