---
layout: post
title: "[Git/Gerrit] Git and Gerrit server hooks"
category: git
tags: [git,gerrit]
---
# [Git/Gerrit] Git and Gerrit server hooks #


### What are git/gerrit hooks? ###
Hook is user’s custom action fired, when git-specific event occurred. Git offers a set of hooks available both on client and server side. In this post, we will focus on server-side hooks for git and gerrit. Unfortunately, gerrit **does not** accept built-in git hooks. Luckily, it offers reach documentation with description of custom gerrit events.


### Git server hooks ###

#### Git offers three types of hooks on server side: ####

* **pre-receive**: operations prepared before actual push is made.

* **post-receive**: operations prepared after actual push is made.

* **update**: similar to pre-receive. The main difference is way of hook execution. While pre-receive is executed just once (no matter how many branches you want push to), update is executed per branch. 

As an example, we are going to create **post-receive** hook. The role of our hook is to notify jenkins server about new push. We retrieve branch name and use CURL to pass url with specific parameters. 

First of all, we go to **hooks** directory in our git server repository. If there’s file called **post-receive.sample** we change it's name to **post-receive** and set proper access mode, if needed: 

{% highlight bash %}
sudo rename post-receive.sample post-receive
sudo chmod a+rx post-receive
{% endhighlight %}

Let’s edit it. Type `sudo nano post-receive` in terminal (fell free to use another text editor), and add following lines:
{% highlight bash %}
# Trigger jenkins with info about new git change
while read oldrev newrev refname
do
    	branch=$(git rev-parse --symbolic --abbrev-ref $refname)
    	curl http://192.168.1.109:8002/git/notifyCommit?url=git@192.168.1.109:repo.git&branches=$branch
done
{% endhighlight %}

After save, we can test our hook in action. On our client machine, I’ve changed one file in my repository:

{% highlight bash %}
git status
git add --all
git commit -m “testing post-receive hook”
git push -u origin master
{% endhighlight %}

After push operation, [I’m checking build queue on jenkins]({{ site.url }}/images/posts/jenkins.png)

**It works! Right after push, new build was triggered.**


### Gerrit server hooks ###
Gerrit offers a range of hooks, connected with code review server operations, like: change merged, comment added, or change abandoned. As an example we will take a look at **change-merged** and **ref-updated** hooks. 

* **change-merged** is used in case of submitting path set into git repository. 
* **ref-updated** is executed in case of any push changes to git repository (if `git push` is used directly, instead of `git review`). 

First of all, we have to find place where hooks are stored. According to gerrit documentation, it’s in **$site_path/hooks** dir. Of course, we can store it in another location. To do so, we edit gerrit config file. In my case, it’s `/home/mody/gerrit/etc/gerrit.config`. At the end of the file, I add:

{% highlight bash %}
[hooks]
    	path = /var/git/repo.git/hooks
{% endhighlight %}

Let’s go to our hooks dir and create `change-merged` and `ref-updates` files.

* **ref-updated**

{% highlight bash %}
ref-updated --oldrev <old rev> --newrev <new rev> --refname <ref name> --project <project name> --submitter <submitter>
{% endhighlight %}

{% highlight bash %}
#!/bin/sh
curl http://192.168.1.109:8002/git/notifyCommit?url=git@192.168.1.109:repo.git&branch=$3
{% endhighlight %}

* **change-merged**

{% highlight bash %}
change-merged --change <change id> --change-url <change url> --project <project name> --branch <branch> --submitter <submitter> --commit <sha1>
{% endhighlight %}

{% highlight bash %}
#!/bin/sh	
curl http://192.168.1.109:8002/git/notifyCommit?url=git@192.168.1.109:repo.git&branch=$4
{% endhighlight %}

The first snippet for each of files is gerrit documentation for specific hook. Second one is file content. In case of **ref-updated**, `--refname` variable is used ($3), in case of **change-merged** `--branch` variable is used ($4) 

Remember to save file with correct access rights:

{% highlight bash %}
sudo chmod a+rx ref-updated
sudo chmod a+rx change-merged
{% endhighlight %}

After work is completed, I restart my gerrit server:

{% highlight bash %}
sudo ./gerrit.sh restart
{% endhighlight %}

Let’s test it. If no errors occurred, our gerrit will send notifications properly.
Any errors will be collected in error_log file in `<YOUR_GERRIT_HOME>/logs` folder.