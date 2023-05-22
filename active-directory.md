---
layout: default
---
<section id="intro">
    <div class="flex-row-between">
        <a href="{{ site.url }}{{ site.baseurl }}/2023/03/28/active-directory-project">« back</a>
        <button title="Change theme" id="theme-toggle" onclick="modeSwitcher()">
            <div></div>
        </button>
    </div>
</section>

Table of Contents
 - [Initial Setup](#initial-setup)


# Initial Setup
I won't go over setting up every single little thing - just the ones that I think is very important to
note. The environment will still match the topology.

## Domain Controllers
I want to be able to use as much PowerShell as possible so I will be installing the AdminToolbox module.

<br>![ATB](/assets/active-directory/admintoolbox.png)
<br>
Next comes the services that will be enabled that can be used by those authenticated to the domain. 
