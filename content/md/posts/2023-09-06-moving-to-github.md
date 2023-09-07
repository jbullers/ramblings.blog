{
:title "Moving to GitHub"
:toc true
:description "Resurrecting my ancient GitLab blog on GitHub."
}
I set up this blog years back on GitLab. At the time, I was using self-hosted GitLab at work, and so decided to use the web version for some personal projects. One of those projects was my writing, which originally was just hosted in a repository so I could easily share with others (primarily co-workers). I used the wiki feature to organize everything, and it worked well enough.

Some time later, I learned about [Gatsby](https://www.gatsbyjs.com/) for static site generation and decided to give it a shot by creating a blog. It was easy enough to set up with GitLab Pages, but I found customization and styling a bit difficult; Gatsby assembles everything as React components. Being unfamiliar, it felt like a bit of a treasure hunt through components to find out what exactly it was that I needed to change to get the results I wanted.

My blog sat dormant since then, the result of both a lack of time and motivation. Recently, I've been getting more into Clojure (more on that in the future), and I discovered [Cryogen](http://cryogenweb.org/) and read a few blogs that were built with it. It looked promising, so I thought it might be fun to reboot my blog with it.

I also decided to put the resurrected blog on GitHub instead of GitLab. Why? Developers I know and work with on occasion (namely my lovely wife and sister-in-law) have accounts on that platform, and all the open source projects I use are there too. I figured I might as well use the platform I frequent rather than split things across multiple platforms. As much as I enjoyed GitLab, I just didn't feel that there was enough draw to keep my projects hosted there.

Cryogen felt a lot easier to style than Gatsby, and a lot more lightweight: it's just vanilla CSS (and Bootstrap) along with Selmer templates. I found it easy to navigate the project structure, and all of my struggles were pretty much confined to just not being very proficient with CSS. I started with the default blue theme (which looks nice and clean) and fiddled with it to apply my favourite colour palette, Nord. I'm pretty happy with the results so far.
