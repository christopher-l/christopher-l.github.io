---
layout: post
title:  "Downloading Manga for E-Book Readers"
date:   2017-02-19
categories:
---
I finally got myself a new e-book reader with a high enough resolution
to properly display manga.
However, while these are pretty accessible for reading on the web on
sites like [mangareader.net](http://www.mangareader.net), it turned out
to be surprisingly cumbersome to actually get these in a format an
e-book reader can show.

## Downloading images

After some not-too-successful time with freeware tools for Windows, I
finally recollected and went for simple scripts on Github.
The one that looked most promising was
[mangareader-to-ebook](https://github.com/saturngod/mangareader-to-ebook)
-- a collection of Python scripts for downloading manga and converting
them to the EPUB or PDF format.
Given a URL on mangareader or similar portals, the `mdl.py` script
attempts to download all images from the given chapter until the last
one that is available, for example

{% highlight bash %}
python2 mdl.py "http://www.mangareader.net/shingeki-no-kyojin/89" ./jpegs
{% endhighlight %}

So I let it do its thing for a while, only to discover that the only
thing it was downloading and putting a `.jpg` suffix on was 403 error
messages.
After some trial and error, I figured out that mangareader doesn't seem
to like download attempts without a `User-Agent` set in the HTTP header.
So I went and changed some lines in the download script and issued a
[pull request](https://github.com/saturngod/mangareader-to-ebook/pull/6),
that should do the trick.

## Converting to e-book format

The
[mangareader-to-ebook](https://github.com/saturngod/mangareader-to-ebook)
repo also contains scripts for conversion to PDF and EPUB, but I wasn't
quite satisfied with the results:
In both formats, it would always produce something with a more or less
fixed page size and a lot of white space around the image.
Once more, Imagemagick turned out to be the solution.
It provides the convert binary, which can take a number of images and
produce a PDF with a page per image that is exactly the size of the
image.

{% highlight bash %}
convert ./jpegs/89/* "pdfs/Chapter 89.pdf"
{% endhighlight %}

Now we're only missing a way to automate this process and give the
files a better filename.
Of course, Bash got us covered:

{% highlight bash %}
#!/usr/bin/bash

imagepath="./jpegs"
outpath="./pdfs"

main() {
    for i in {1..3} ; do
        command="convert $imagepath/$i/* \"$outpath/[$i] ${names[i]}.pdf\""
        echo "$command"
        eval "$command"
    done
}

names[1]="To You, 2,000 Years from Now (二千年後の君へ Ni Sen Nen Go no Kimi e)"
names[2]="That Day (その日 Sono Hi)"
names[3]="Night of the Disbanding Ceremony (解散式の夜 Kaisanshiki no Yoru)"
# and so on

main
{% endhighlight %}

