---
layout: default
title: Time Is Not Metadata
---

{% capture paper %}{% include_relative README.md %}{% endcapture %}
{{ paper | replace: 'IMPLEMENTATION.md', 'implementation.html' }}
