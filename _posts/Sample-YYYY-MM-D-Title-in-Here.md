---
layout: single
header:
  overlay_image: /assets/images/banner-005.jpg
  overlay_filter: 0.6
toc: true
toc_depth: 6
toc_label: Table of Contents
last_modified_at: YYYY-MM-DDTHH:MM:SS+01:00
categories:
  - categorie1
  - categorie2

tags:
  - tag1
  - tag2
  - tag3
---

#### images (One Up)
```yaml
<p align="center"><a href="/_posts/attachments/image1.png"><img src="/_posts/attachments/image1.png"></a></p>
```

#### images (Two Up)
```yaml
<figure class="half">
    <a href="/_posts/attachments/image1.png"><img src="/_posts/attachments/image1.png"></a>
    <a href="/_posts/attachments/image1.png"><img src="/_posts/attachments/image1.png"></a>
</figure>
```

#### images (Tree Up)
```yaml
<figure class="third">
	<img src="/_posts/attachments/image1.png">
	<img src="/_posts/attachments/image2.png">
	<img src="/_posts/attachments/image3.png">
	<figcaption>Caption describing these three images.</figcaption>
</figure>

```

#### links
```sh
[Text to click][Link]              ## this is in text clickable, in text
[Link]: https://www.site.com/      ## this is the link source, add in the bottom
```

