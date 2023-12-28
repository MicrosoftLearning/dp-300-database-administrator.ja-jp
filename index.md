---
title: オンラインでホスティングされている手順
permalink: index.html
layout: home
---

# データベース管理の演習

これらの演習では、Microsoft コース [DP-300: Microsoft Azure SQL ソリューションの管理](https://docs.microsoft.com/training/courses/dp-300t00)がサポートされています。

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %}
| モジュール | 演習 |
| --- | --- | 
{% for activity in labs %}| {{ activity.lab.module }} | [{{ activity.lab.title }}{% if activity.lab.type %} - {{ activity.lab.type }}{% endif %}]({{ site.github.url }}{{ activity.url }}) |
{% endfor %}

