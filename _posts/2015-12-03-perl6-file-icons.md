---
title:  "Perl 6 File Icons in Atom Editor"
date:   2015-12-03 11:26:00
description: Get nice Camelia file icons for your Perl 6 files in Atom Editor.
---

This morning, I decided to spend some time trying to get file icons for Perl 6 files.  None of the font packages that file-icons includes have a Perl 6 icon so I would need to take the Camelia logo and turn it into a small icon.

Being a complete SVG noob (and without even trying to learn), I tried to edit the svg xml directly.  I adjusted the size, changed some metadata, and then completely lost myself looking at all the paths.  At this point I realized that I was doing something wrong as the image was not showing in the thumbnail of Gnome Files.

After fumbling my way through Inkscape for a bit, I was lucky enough to get pinged by Juerd on irc that he already had a mono svg of Camelia.

![Juerd's monochrome Camelia](http://juerd.nl/tmp/Camelia_mono_1path.svg)

At this point, I began to look up how to turn that into a font as Atom Editor is just a glorified web browser which means I would need some kind of web font (specifically, `woff`).  I found a fantastic free site called [icomoon](https://icomoon.io/app) that lets you turn custom svg files into icon fonts.  So I converted that and dropped the folder into my `.atom` folder under `stylesheets`.  At this point I struggled a bit to understand how to import icomoon's css into my `styles.less` but after some help on Slack, I had it working!

![Example shot of camelia file icons](https://pbs.twimg.com/media/CVUCgQkUsAEo41l.png:large)

Check out my [tweet](https://twitter.com/MadcapJake/status/672446473682837505) for a larger image. If you're interested in having these lovely file icons, here's how you do it:

```shell
apm install file-icons
cd ~/.atom/stylesheets
git clone https://github.com/MadcapJake/perl6-atom-file-icon camelia
sed -i "s|/home/jrusso|$HOME|" ~/.atom/stylesheets/camelia/style.css
atom ~/.atom/styles.less
```
Then add this:

```
@import "packages/file-icons/styles/colors";
@import "packages/file-icons/styles/items";
@import "packages/file-icons/styles/icons";
@import (less) "stylesheets/camelia/style.css";

.octicon-package {
  content: "\f0c4"
}

@{pane-tab-selector}, .icon-file-text {
  &[data-name="META6.json"]:before { .octicons; .octicon-package !important; }
}
@{pane-tab-selector}, .icon-file-text {
  &[data-name="META.info"]:before { .octicons; .octicon-package !important; }
}
@{pane-tab-selector}, .icon-file-text {
  &[data-name$=".pl6"]:before { .camelia; .im-camelia; .light-green; }
}
@{pane-tab-selector}, .icon-file-text {
  &[data-name$=".pm6"]:before { .camelia; .im-camelia; .light-blue; }
}
@{pane-tab-selector}, .icon-file-text {
  &[data-name$=".t"]:before { .camelia; .im-camelia; .light-yellow; }
}
```

That should do it!  You can change the colors if you like, too.  Of course, keep in mind that changing the `.t` extension will overwrite any perl5 file icon (I think it's a camel).

Enjoy!
