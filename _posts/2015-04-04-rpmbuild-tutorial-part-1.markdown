---
layout: post
comments: true
title:  "rpmbuild tutorial - how to build rpm packages"
date:   2015-04-04 20:03:43
categories: rpm rpmbuild 
---

Welcome to the first part article on how to build rpm packages. Here I will walk you through how to build a rpm package and how to work with the tools you will need. Let me just first start off with saying that I don't consider myself an expert at rpm packaging but blogging about a topic forces you to graps the subject you are writing about better. With that being said this first part will cover the bascis like installing rpm tools, the structure of a spec file, macros and lastly a simple rpm build. In part two my ambition is to take the things I wrote about here to package a real world application into an rpm.

<!-- more -->

**GETTING STARTED**

Let's start off! First just a word of caution, do not use the root user as the account which you build packages with. It's much safer to build the packages with a user that do not have root permission if you mess something up. With that out of the way, we'll start off grabbing the stuff we need:

{% highlight sh %}
[bob@vagrant-centos65 ~]$ sudo yum install -y rpmdevtools rpmlint
{% endhighlight %}

Next we'll set up the folder structure with the rpmdev-setuptree command. It will create tbe rpmbuild folder and it's subfolders as shown below:

{% highlight sh %}
[bob@vagrant-centos65 ~]$ rpmdev-setuptree
[bob@vagrant-centos65 ~]$ tree rpmbuild/
rpmbuild/
├── BUILD
├── RPMS
├── SOURCES
├── SPECS
└── SRPMS
{% endhighlight %}

Here's a short description of each of the directories:

* ```BUILD``` - this is the folder where all of the files go which are created during a build of the package you want to create
* ```RPMS / SRPMS``` - If your build is successful this is the folder there the final rpm file will be created. The SRPMS folder only contains source rpm.
* ```SOURCES``` - This is the folder where you want to put the source tar file or/and additional files such as init.d/systemd files.
* ```SPECS``` - The SPECS folder is where your spec-file is located. A spec file is basically a set of instructions on how to build the rpm package. I will explain the layout of the spec-file in detail.


**THE SPECFILE**

If there was one thing that you should know about creating a rpm it's how to write the spec file. As I wrote earlier a spec file consists of instructions on how to package the software into an rpm and how it should be installed on the OS. Not only that but also information about the package/software such as version and licence.

Move to the SPECS folder and use the rpmdev-newspec utility to create a specfile template. 

{% highlight sh %}
[bob@vagrant-centos65 ~]$ cd rpmbuild/SPECS/
[bob@vagrant-centos65 SPECS]$ rpmdev-newspec
Skeleton specfile (minimal) has been created to "newpackage.spec".
[bob@vagrant-centos65 SPECS]$ mv newpackage.spec simple-script.spec
{% endhighlight %}


We will build a rpm package out of this specfile, but first just let's figure out the different parts and how the specfile works. Here is the content of the spec-file we recently created. Let's discuss each of the entries:


{% highlight sh %}
Name:
Version:
Release:        1%{?dist}
Summary:

Group:
License:
URL:
Source0:

BuildRequires:
Requires:

%description


%prep
%setup -q


%install
rm -rf $RPM_BUILD_ROOT
make install DESTDIR=$RPM_BUILD_ROOT


%clean
rm -rf $RPM_BUILD_ROOT


%files
%defattr(-,root,root,-)
%doc
{% endhighlight %}


* ```Name, Version, Release``` - Should be pretty straight forward. Name the package, it's version and it's release (dist is a macro that will translate to the red-hat flavour dist you build on).
* ```Summary, %description```- Easy stuff as well. Summary should be one sentence describing the software while %description should be longer and more descriptive.
* ```Group, Licence, URL, Source0 ``` - Same here, this is also pretty basic stuff. Group is used to categorize the package. If unsure take a look at /usr/share/doc/rpm-4.8.0/GROUPS to see the full list of available groups. Next we have licence which should be included, if for example you are packaging Elasticsearch you should write that corresponding licence in your spec-file. 
Source0 is important, here you write the name of the source tar file in the SOURCES directory. You can write multiply Source in the specfile like this; Source0, Source1, Source2 and so on. You would do this if you want to point to a startscript, logrotate or libs that you don't want to inclide in the source tar file.
* ```BuildRequires, Requires ``` - BuildRequires is the category you need to list the packages that the software you are packaging requires at build time, whereas Requires is what your rpm requires at runtime. For example, if we're packaging tomcat we need to list java as a requirement for the runtime, but we dont need any special BuildRequires as tomcat does not need to be compiled.
* ```%files``` - %files is an important field because here we specify every folder or file(-s) we will install on the computer. The %defaultattr macro sets the permission of the file(-s), in this case root. Use %doc to highligh which files are READMEs.

You might noticed we didn't cover the %prep and %setup definitions. We will cover them shortly but first let's talk about macros.

**MACROS**

If you have taken a look at a spec file before you'll surely have noticed that the spec file are littered with macros, it's the variable-looking stuff thats starts with a %-character. There are a lot of built in macros (such as %{_sysconfigdir}) but you can also define your own macros, such as a global variable. If unsure about what a macro translates to (which can be kind of confusing) I recommend using the rpm --eval command like shown below:

{% highlight sh %}
[bob@vagrant-centos65 ~]$ rpm --eval %{_sysconfdir}
[bob@vagrant-centos65 ~]$ /etc
{% endhighlight %}

The scope of this article is not to describe every kind of macro, but the three kinds are the most uselful:

* Built-in macros, including the following useful directories (use these macros instead of hardcoding paths):
{% highlight sh %}
%_prefix /usr
%_exec_prefix %{_prefix}
%_bindir %{_exec_prefix}/bin
%_sbindir %{_exec_prefix}/sbin
%_libexecdir %{_exec_prefix}/libexec
%_datadir %{_prefix}/share
%_sysconfdir %{_prefix}/etc
%_sharedstatedir %{_prefix}/com
%_localstatedir %{_prefix}/var
%_libdir %{_exec_prefix}/lib
%_includedir %{_prefix}/include
%_oldincludedir /usr/include
%_infodir %{_prefix}/info
%_mandir %{_prefix}/man
{% endhighlight %}


* User-defined macros. To define your own macro, that you can later use in as macro/variable in the specfile, you must use the "define" keyword. For example: 

{% highlight sh %}
%define major_version 1
%define minor_version 4
%define micro_version 4
%define prog_name     elasticsearch
%define prog_dir      /var/lib/elasticsearch
{% endhighlight %}

You could then combine these macros you'd defined with for example:

{% highlight sh %}
Version: %{major_version}.%{minor_version}.%{micro_version}
{% endhighlight %}


* Spec-file specifc macros, these control the flow of executing of the spec-file.

* ```%pre```	   - This is where code is run before the install scripts run.
* ```%preun```     – The section where code is executed prior to uninstall.
* ```%post```      – Section for code to be executed after installation.
* ```%postun```    – Section for code to be executed after uninstallation.
* ```%files```     - This section declares which files and directories are owned by the package, and thus which files and directories will be placed into the binary RPM.


**REAL WORLD EXMAPLE**

Okey, enough with the theory *phu*. Let's build a simple but real world rpm package. Just follow along on the command-line:

{% highlight sh %}
[bob@vagrant-centos65 ~]$ pwd
/home/bob
b@vagrant-centos65 ~]$ mkdir myscript-1.0
[bob@vagrant-centos65 ~]$ vim myscript-1.0/myscript.sh
#!/bin/bash
echo "Hello from bash. Zsh is for hipsters."
[bob@vagrant-centos65 ~]$ tar czvf rpmbuild/SOURCES/myscript-1.0.tar.gz myscript-1.0/
myscript-1.0/
myscript-1.0/myscript.sh
[bob@vagrant-centos65 ~]$ chmod +x myscript-1.0/myscript.sh
[bob@vagrant-centos65 ~]$ rpmdev-newspec rpmbuild/SPECS/myscript.spec
Skeleton specfile (minimal) has been created to "rpmbuild/SPECS/myscript.spec".
[bob@vagrant-centos65 ~]$ vim rpmbuild/SPECS/myscript.spec


Name:           myscript
Version:        1.0
Release:        1%{?dist}
Summary:        Small epic script but not really usable

Group:          Utilities
License:        GPL
Source0:        myscript-%{version}.tar.gz
BuildArch:      noarch

%description
Small epic script that prints some stuff

%prep
%setup -q


%install
rm -rf $RPM_BUILD_ROOT
install -d $RPM_BUILD_ROOT/opt/myscript
install myscript.sh $RPM_BUILD_ROOT/opt/myscript/myscript.sh


%clean
rm -rf $RPM_BUILD_ROOT


%files
%defattr(-,root,root,-)
/opt/myscript/myscript.sh
%doc

%changelog

[bob@vagrant-centos65 ~]$ rpmbuild -bb rpmbuild/SPECS/myscript.spec
Executing(%prep): /bin/sh -e /var/tmp/rpm-tmp.EdXMOM
+ umask 022
+ cd /home/bob/rpmbuild/BUILD
+ LANG=C
+ export LANG
+ unset DISPLAY
+ cd /home/bob/rpmbuild/BUILD
+ rm -rf myscript-1.0
+ /bin/tar -xf -
+ /usr/bin/gzip -dc /home/bob/rpmbuild/SOURCES/myscript-1.0.tar.gz
+ STATUS=0
+ '[' 0 -ne 0 ']'
+ cd myscript-1.0
+ /bin/chmod -Rf a+rX,u+w,g-w,o-w .
+ exit 0
Executing(%install): /bin/sh -e /var/tmp/rpm-tmp.Zw4n1D
+ umask 022
+ cd /home/bob/rpmbuild/BUILD
+ '[' /home/bob/rpmbuild/BUILDROOT/myscript-1.0-1.el6.x86_64 '!=' / ']'
+ rm -rf /home/bob/rpmbuild/BUILDROOT/myscript-1.0-1.el6.x86_64
++ dirname /home/bob/rpmbuild/BUILDROOT/myscript-1.0-1.el6.x86_64
+ mkdir -p /home/bob/rpmbuild/BUILDROOT
+ mkdir /home/bob/rpmbuild/BUILDROOT/myscript-1.0-1.el6.x86_64
+ cd myscript-1.0
+ LANG=C
+ export LANG
+ unset DISPLAY
+ rm -rf /home/bob/rpmbuild/BUILDROOT/myscript-1.0-1.el6.x86_64
+ install -d /home/bob/rpmbuild/BUILDROOT/myscript-1.0-1.el6.x86_64/opt/myscript
+ install myscript.sh /home/bob/rpmbuild/BUILDROOT/myscript-1.0-1.el6.x86_64/opt/myscript/myscript.sh
+ /usr/lib/rpm/check-rpaths /usr/lib/rpm/check-buildroot
+ /usr/lib/rpm/redhat/brp-compress
+ /usr/lib/rpm/redhat/brp-strip /usr/bin/strip
+ /usr/lib/rpm/redhat/brp-strip-static-archive /usr/bin/strip
+ /usr/lib/rpm/redhat/brp-strip-comment-note /usr/bin/strip /usr/bin/objdump
+ /usr/lib/rpm/brp-python-bytecompile
+ /usr/lib/rpm/redhat/brp-python-hardlink
+ /usr/lib/rpm/redhat/brp-java-repack-jars
Processing files: myscript-1.0-1.el6.noarch
Requires(rpmlib): rpmlib(CompressedFileNames) <= 3.0.4-1 rpmlib(FileDigests) <= 4.6.0-1 rpmlib(PayloadFilesHavePrefix) <= 4.0-1
Checking for unpackaged file(s): /usr/lib/rpm/check-files /home/bob/rpmbuild/BUILDROOT/myscript-1.0-1.el6.x86_64
warning: Could not canonicalize hostname: vagrant-centos65.vagrantup.com
Wrote: /home/bob/rpmbuild/RPMS/noarch/myscript-1.0-1.el6.noarch.rpm
Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.4pDBrn
+ umask 022
+ cd /home/bob/rpmbuild/BUILD
+ cd myscript-1.0
+ rm -rf /home/bob/rpmbuild/BUILDROOT/myscript-1.0-1.el6.x86_64
+ exit 0

[root@vagrant-centos65 ~]# rpm -ivh /home/bob/rpmbuild/RPMS/noarch/myscript-1.0-1.el6.noarch.rpm
Preparing...                ########################################### [100%]
   1:myscript               ########################################### [100%]
[root@vagrant-centos65 ~]# /opt/myscript/myscript.sh
Hello from bash. Zsh is for hipsters.
{% endhighlight %}











