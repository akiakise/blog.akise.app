---
title: 小伙伴们
subtitle: 总有一天，地球发出的光将穿过那些星座，在每一个星座中，我们将再次年轻
robots: noindex,nofollow
sitemap: false
comment_title: 快来交换友链吧～
header: false
banner: /img/website/friends.png
menu_id: social
leftbar: [social, recent]
---

{% friends api:https://raw.githubusercontent.com/akiakise/issues-json-generator/refs/heads/output/v2/data.json %}

{% note 如果友链失联了？ 友链如果长期失联，可能会被取消！届时 GitHub Issue 的标签也会更新，如果您的网站恢复了，请在申请友链时创建的 [Issue](https://github.com/akiakise/issues-json-generator/issues) 中评论告知。 %}

{% quot 如何交换友链 icon:hashtag %}

您的网站需要满足以下全部条件：
{% checkbox checked:true 合法的、非营利性、无商业广告、无木马植入 %}
{% checkbox checked:true 有实质性原创内容的 HTTPS 站点，发布过至少 5 篇原创文章，内容题材不限 %}
{% checkbox checked:true 有独立域名，非免费域名 %}
{% checkbox checked:true 博客已运行至少半年，非刚搭建好 %}


如果您已经满足条件，请按照如下步骤自助提交友链：

{% timeline %}
<!-- node 第一步: 新建 Issue -->
新建 [GitHub Issue](https://github.com/akiakise/issues-json-generator/issues) 并按照新 Issue 模板填写并提交。

<!-- node 第二步: 添加本站友链 -->
请将本站添加到您的友链中:

```yaml
title: AA博客
url: https://blog.akise.app
avatar: https://blog.akise.app/img/website/avatar.jpg
```

<!-- node 第三步: 等待站长操作 -->
我在收到新 Issue 后会结合上边的标准评估是否添加，如果添加会在 Issue 上新增 `active` 标签，标签新增完成后大约 3 分钟内友链就会生效。

如果您需要更新自己的友链，请直接修改 `Issue` 内容，大约 3 分钟内生效，无需等待博客重新部署。

{% endtimeline %}