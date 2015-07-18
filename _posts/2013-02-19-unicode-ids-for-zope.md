---
title: Allowing unicode ids in Zope
description:
  "Back when Zope was written Python did not support unicode. This is still
  reflected in the policy for object ids in Zope: they must be pure ASCII.
  I recently started wondering what it would bring Zope a bit closer to the
  future and make it support UTF-8 ids. It turns out that this is doable
  without too much effort."
---
I recently had someone ask me why certain files were not showing up in a
Reflecto instance. This turned out to be caused by the use of non-ASCII
filenames. Reflecto was doing the correct thing and skipped those files since
those would not result in valid object ids in Zope. Since that is a not very
user friendly I started poking around in Zope to see what would need to be done
to fix that. It turns out that the Zope publisher and all URL-handling code
already does the right thing: URLs are properly quoted and unquoted in all the
necessary places. This allowed me to simplify the valid id checks in Reflecto,
and suddenly those non-ASCII files showed up and worked fine. Unicode URLs just
worked:

![Screenshot of Reflecto](http://www.wiggy.net{{ site.static.utf8id_reflecto_png.url }} "Reflecto showing non-ASCII filename")

After having finished this I started wondering what it would take to loosen
the id rules for Zope itself. Zope has one golden rule: object ids (both the
traditional `id` attribute and the newer `__name__` attribute **must** be str
instances. Using that as a basis I started modifying code to allow any valid
UTF-8 encoded alphanumerics instead of just ASCII alphanumerics. Doing this
revealed a few things:

* Even though Zope does not allow it some packages did use unicode ids because
  they were using ZTK packages designed for (what used to be) Zope3 such as
  `zope.container`.
* Simple browser pages defined via ZCML accidentily got a unicode `__name__`.
* There are far too many code paths for validating object ids and at least
  two implementations for the actual checks: for unknown reasons Plone decided
  to reimplement this.

Where possible I fixed the use of unicode ids in the original packages. In order
to test the use of UTF-8 ids I created a new
[`experimental.utf8id`](https://github.com/wichert/experimental.utf8id)
package which applies some careful monkeypatches to loosen the id tests and
modify default normalized and name choosers. With that package installed
Plone seems to work fine with non-ASCII object ids:

![Screenshot of Plone](http://www.wiggy.net{{ site.static.utf8id_plone_png.url }} "Plone showing off UTF-8 object ids")

Currently this is not for the faint of heart: it requires unreleased versions
of Zope, plone.portlets, plone.app.portlets and experimental.utf8id itself. It
is however a promising start. As far as I can see there is no reason we can not
start making these kind of changes for Zope 2.14.

If you want to give this a try [grab
`experimental.utf8id`](https://github.com/wichert/experimental.utf8id) and
follow its installation instructions.
