---
layout: post
title:  "Insert millions of products to load test Odoo POS"
date:   2019-07-23 00:16:33 +0700
categories: odoo
---

Odoo comes with a prebuilt POS module. Under most circumstances, the module would behave nicely and be sufficient for basic usage. However, some retailers have very large assortment of products, which slows the POS screen down significantly. Thus it is important for users to calibrate their need versus the system's capability before adopting POS Odoo. In the scope of this post, we are interested in how far we can push the system until the point of non-functional. 

To do that, we will be loading Odoo's database with an increasing number of products and measure degradation in POS screen's response time if any. 

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
