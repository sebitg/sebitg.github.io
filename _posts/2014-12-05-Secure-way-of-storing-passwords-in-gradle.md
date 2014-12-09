---
layout: post
title: "[Gradle] Secure way of storing passwords"
category: gradle
tags: [grade]
---
# [Gradle] Secure way of storing passwords #

### Problem ###
Using passwords in build.gradle file can be cumbersome. The question is, where to store passwords? Typing it directly in build.gralde, or gradle.properties is bad idea (especially when working with git - accidental commit of properties etc.). Fortunatelly, there is good OOTB (Out-of-the-box) solution in gradle, which helps us to solve that problem.

### Gradle init script ###
Gradle offers initialization script mechanism. It is executed before build starts. There are few ways of include init script into our build:

* Specify **init.gradle** file anywhere in file system. Then, type in console:

{% highlight bash %}
gradle --init-script=/path/to/your/script/init.gradle -q someTask
{% endhighlight %}

* Specify **init.gradle** file in **YOUR_HOME_DIR/.gradle**. Init script will be loaded automatically when executing gradle tasks.
* Specify file ended with **.gradle** in **GRADLE_HOME/.gradle**. Init script will be loaded automatically when executing gradle tasks.

You are free to load any dependencies you need in your init sript. A typical way to do that is the following snippet:
{% highlight groovy %}
import com.btcgroup.gradleplugin.utils.*

initscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath group: 'com.btcgroup', name: 'gralde-plugin-utils', version: '1.1.+'
    }
}

def list = Util.getProjects()
{% endhighlight %}

### Example ###
As an example, we will create external properties loader. Properties contain passwords, so the best idea is to store it in secure place. For the purpose of this example, I will store my properties file in the same place, as **init.gradle** script, that's: **YOUR_HOME_DIR/.gradle**. 

######init-settings.properties######
{% highlight groovy %}
archiva.url=http://localhost:8080/repository/myproject/
archiva.username=xxxxx
archiva.password=xxxxx
{% endhighlight %}

######init.gradle######
{% highlight groovy %}
import java.util.Properties
import java.io.*

allprojects {
	println "[INIT SCRIPT] Initializing script START..."
	Properties props = loadExternalProps()
	beforeEvaluate { project ->
		if(props!=null) {
			println "[INIT SCRIPT] Loading properties for project " + project.name
			project.ext.externalProps = props
		}
	}
}

def loadExternalProps() {
	String userGradleDir = System.getProperty("user.home") + "/.gradle/"
	File file = new File(userGradleDir + "init-settings.properties")
	if(!file.exists())
		return null
	Properties props = new Properties()
	props.load(new FileInputStream(file))
	return props
}
{% endhighlight %}

#####What's happening there?#####
* First of all, we want to evaluate our logic to all projects builded at the time. That's why we use `allprojects` clause. 
* Right after that, we use `beforeEvaluate` callback for specific project. Another option is to use `afterEvaluate` callback, but in case of properties loading, `beforeEvaluate` is the best choice.
* Including our properties. We do it by adding `externalProperties` object to `ext` object of `project`: `project.ext.externalProps = props`

#####How to use it in our build.gradle scripts?#####
It's really simple, and refers to use ***externalProperies*** object inside ***build.gradle***. Take a look at this example:
{% highlight groovy %}
repositories {
    mavenCentral()
}
dependencies {
}
...
task someSampleTask << {
    println externalProps["archiva.username"]  //Example of use
}
{% endhighlight %}