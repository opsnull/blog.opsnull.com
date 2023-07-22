---
layout: simple
menu:
  footer:
    name: Now
    identifier: "now"
    weight: 20
---

{{< typeit tag=h2 lifeLike=true >}}
这里显示我最近的`活动`。
{{< /typeit >}}

{{< timeline >}}

{{< timelineItem icon="github" header="header" badge="badge test" subheader="subheader" >}}
学习 eBPF。 
{{</ timelineItem >}}


{{< timelineItem icon="code" header="Another Awesome Header" badge="date - present" subheader="Awesome Subheader">}}
支持 html 代码。
<ul>
  <li>Coffee</li>
  <li>Tea</li>
  <li>Milk</li>
</ul>
{{</ timelineItem >}}
{{</ timeline >}}
