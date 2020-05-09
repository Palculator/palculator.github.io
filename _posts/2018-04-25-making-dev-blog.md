---
layout: post
title: Making a dev blog
---

### Setup

Setting up this site was actually pretty easy, thanks to [Jekyll][1] being a
very simple framework and [GitHub pages][2] being available for free. Designing
it took a bit more work, but, thankfully, I was able to find am MIT-licensed
blog theme called [Hyde][3] that I could easily adapt more to my liking. There
were some hiccups with the theme's code being written for the now-outdated
Jekyll 2, but migrating was easy enough. After the base theme was working in
`jekyll serve` I set about changing it a bit to be more to my liking, but no
huge changes were necessary.

[1]: https://jekyllrb.com/
[2]: https://pages.github.com/
[3]: https://github.com/poole/hyde

### Usage

What all this means is that now I can easily write posts on this page by simply
writing a markdown-formatted file in the `_posts` folder of [my repository][6].
They only have to follow the naming convention of `YYYY-MM-DD-title.md` and 
declare their layout to be `post` in the frontmatter to be recognised.
Everything else is done by Jekyll and the templates in the repository.
Unfortunately, syntax highlighting isn't really part of the Markdown standard,
but it can be added by enclosing code blocks within `highlight` tags in the
post file. Python code, for example, will then look pretty, like this:

{% highlight python %}
import numpy as np
spam = np.array(range(4096))
for i in spam:
    print(i)
{% endhighlight %}

And, of course, the actually standard features of [Markdown][4] are all
supported, so editing in general is fairly easy. I'll look into adding some
bells and whistles like LaTeX formatting ([apparently isn't too hard][5]) but,
for now, I'm pretty happy with how smooth all of this went.

[4]: https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet
[5]: https://zishuaiz.github.io/blog/how-to-enable-mathjax-in-github-pages
[6]: https://github.com/Palculator/palculator.github.io
