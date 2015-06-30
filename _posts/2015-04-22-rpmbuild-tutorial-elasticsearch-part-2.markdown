---
layout: post
comments: true
title:  "rpmbuild tutorial - Elasticsearch source rpm"
date:   2015-04-22 22:43:43
categories: rpm rpmbuild elasticsearch 
---


**GETTING STARTED**

First let's set up our build rpm environment by installing the following:

{% highlight sh %}
[bob@vagrant-centos65 ~]$ sudo yum install -y rpmdevtools rpmlint
{% endhighlight %}

Then we'll set up the folder structure with the rpmdev-setuptree command. It will create tbe rpmbuild folder and it's subfolders as shown below.

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

In the SPEC folder we create elasticsearch.spec, and in the SOURCES folder we download the tar-file from elasticsearch.


{% highlight sh %}
%global major_version 1
%global minor_version 4
%global micro_version 4

%global bindir /usr/share/elasticsearch
%global confdir /etc/elasticsearch

Name:           elasticsearch
Version:        %{major_version}.%{minor_version}.%{micro_version}
Release:        1
Summary:        A Distributed RESTful Search Engine

Group:          System Environment/Daemons
License:        Apache License Version 2.0
URL:            https://www.elastic.co/
Source0:        https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-%{major_version}.%{minor_version}.%{micro_version}.tar.gz
Source1:        elasticsearch.init

Requires:       java-1.7.0-openjdk
Requires(pre):  shadow-utils

%description
Elasticsearch is a distributed RESTful search engine built for the cloud.

%prep
%setup -qn elasticsearch-%{version}
%{__rm} ./bin/*.bat
%{__rm} ./bin/*.exe
%{__rm} ./lib/sigar/*.dylib
%{__rm} ./lib/sigar/*.dll
%{__rm} ./lib/sigar/*.lib
%{__rm} ./lib/sigar/libsigar*freebsd*
%{__rm} ./lib/sigar/libsigar*solaris*


%build
# empty

%install
%{__mkdir_p} %{buildroot}/%{bindir}
%{__mkdir_p} %{buildroot}/%{confdir}
%{__mkdir_p} %{buildroot}/etc/init.d
%{__cp} -a bin lib %{buildroot}/%{bindir}/.
%{__cp} -a config/*.yml %{buildroot}/%{confdir}/.
%{__cp}  %{SOURCE1} %{buildroot}/etc/init.d/elasticsearch
%{__mkdir_p} %{buildroot}%{_localstatedir}/run/elasticsearch
%{__mkdir_p} %{buildroot}%{_localstatedir}/lib/elasticsearch
%{__mkdir_p} %{buildroot}%{_localstatedir}/log/elasticsearch

%pre
# add elasticsearch user and group
if ! /usr/bin/getent group elasticsearch >/dev/null; then 
  /usr/sbin/groupadd -r elasticsearch
fi

if ! /usr/bin/getent passwd elasticsearch >/dev/null; then  
  /usr/sbin/useradd -g elasticsearch -s /bin/nologin -r -d /usr/share/elasticsearch elasticsearch
fi

%post
/sbin/chkconfig --add elasticsearch

%clean
%{__rm} -rf %{buildroot}

%files
%doc 
LICENSE.txt 
NOTICE.txt 
README.textile

%defattr(-,root,root,-)
%{bindir}
%{confdir}/etc/init.d/elasticsearch
%defattr(-,elasticsearch,elasticsearch,-)
%{_localstatedir}/run/elasticsearch
%{_localstatedir}/lib/elasticsearch
%{_localstatedir}/log/elasticsearch

%changelog
* Tue Jun 30 2015
- Roger W. Built from Elastic binaries

{% endhighlight %}

Lastly build the rpm package:


{% highlight sh %}
[bob@vagrant-centos65 ~]$ rpmbuild -bb rpmbuild/SPECS/elasticsearch.spec
{% endhighlight %}









