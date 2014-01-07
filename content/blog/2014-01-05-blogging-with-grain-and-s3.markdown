---
layout: post
title: "Blogging with Grain and S3"
date: "2014-01-05 18:39"
author:
categories: [grain, aws, s3, route53, cloudfront]
comments: true
published: true
---

I prefer static site generators when it comes to blogging: they're easy to store in version control, and they're pretty bulletproof security-wise. I'd used [Octopress](http://octopress.org/docs/) before, as well as plain [Jekyll](http://jekyllrb.com/), and though I liked the concept, in practice neither worked smoothly: the whole gem infrastructure is kinda messy, and "watch for changes" mode didn't work reliably. So, when I saw a blurb about [Grain](http://sysgears.com/grain/), a static site generator written in Groovy, I investigated and was pleased to see that it (1) had an Octopress theme clone for easy blog setup, (2) was written with (IMO) best-in-class tech choices: Groovy, Guice, and Gradle, and (3) had watch-for-changes that actually worked.

## Grain

This blog uses the [Octopress theme](http://sysgears.com/grain/themes/octopress/) for [Grain](http://sysgears.com/grain/). I chose to [fork (see the varblog branch)](https://github.com/marshallpierce/grain-theme-octopress) the [main octopress theme repo](https://github.com/sysgears/grain-theme-octopress) so that I could more easily incorporate future improvements, rather than starting a new repo using a released version. Especially as Grain matures, you may wish to just take a released version and go from there, but for now using a fork has been fine, and it's let me easily make pull requests as I make improvements that could be generally useful to other users.

I encourage interested readers to go look at [the commits in my fork](https://github.com/marshallpierce/grain-theme-octopress/commits/varblog) to see all the setup steps I took, but I'll point out one in particular. My Linux system used Python 3 by default, which wasn't compatible with the version of Pygments bundled with Grain. So, to change it to look for python 2 first, I added the following to my SiteConfig in the `features` section:

```
    python {
        cmd_candidates = ['python2']
    }
```

## S3 Hosting

Hosting static output in S3 is pretty common. The speed and reliability of S3 is tough to beat, and even though it's non-free, for most people hosting a blog on S3 will cost less than $1 a month.

I first created an S3 bucket named the same thing as the domain (`varblog.org`). I enabled static website hosting for the bucket (using `index.html` as the index document) and set the bucket policy to allow GetObject on every object:

```
{
    "Version":   "2008-10-17",
    "Id":        "Policy1388973900126",
    "Statement": [
        {
            "Sid":       "Stmt1388973897544",
            "Effect":    "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action":    "s3:GetObject",
            "Resource":  "arn:aws:s3:::varblog.org/*"
        }
    ]
}
```

I made a Route 53 hosted zone for `varblog.org` (remember to change your domain's nameservers to be Route 53's nameservers) and set up an alias record for `varblog.org` to point to the S3 bucket.

## Uploading to S3

I created a dedicated IAM user for managing the bucket and gave it this IAM policy:

```
{
    "Version":   "2012-10-17",
    "Statement": [
        {
            "Sid":      "Stmt1388973098000",
            "Effect":   "Allow",
            "Action":   [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::varblog.org/*"
            ]
        },
        {
            "Sid":      "Stmt1388973135000",
            "Effect":   "Allow",
            "Action":   [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::varblog.org"
            ]
        }
    ]
}
```

This allows that user to do everything to the `varblog.org` bucket and its contents, but not to do any other AWS actions. I created an Access Key for the user in the IAM console and used it to configure [`s3cmd`](http://s3tools.org/s3cmd) with `s3cmd --configure -c ~/.s3cfg-varblog.org`. This creates a separate config file which I then reference in the `s3_deploy_cmd` in `SiteConfig.groovy`. This way, even though I'm storing an access key &amp; secret unencrypted on the filesystem, the credentials only have limited AWS privileges, and I'm not conflating this `s3cmd` configuration with other configurations I have. Note that when configuring `s3cmd`, it will ask if you want to test the configuration. Don't bother, as this test will fail: it tries to list all buckets, but this isn't allowed in the IAM user's policy.

At this point, `./grainw deploy` will populate the bucket with the generated contents of the site.

## Other stuff
For Google Analytics and Disqus I simply created new sites and plugged in the appropriate ids in `SiteConfig`. I chose to update the GA snippet template since by default new GA accounts use the "universal" tracker which has a different snippet than good old `ga.js`. If your GA account is old-school, you should be able to leave the template as-is.

Other than that, all I did was tweak some `SASS` in `theme/sass/custom`.

## What's with the name?

If you're not a Linux/Unix user, this blog's name will make no sense. Then again, the rest of this post probably didn't either. The `/var/log` directory is historically where log files have gone on Unix-y systems, and 'blog' is kind of like 'log'. Or, put another way, I thought it was amusing when I registered this domain long, long ago.