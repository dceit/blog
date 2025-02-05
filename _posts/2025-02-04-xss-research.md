---
layout: post
title: Advanced XSS Vectors - Fuzzing Browser Parsing & "MySpace"
---
### Introduction
While doing cross-site-scripting (XSS) research, I've come across many filters and markdown implementations which allow for basic HTML such as `<h1>Hello world!</h1>`, but disallow potentially malicious tags such as `<script>`.

A common approach in these filters involves either blacklisting specific tags or whitelisting safe ones. For example, a filter might strip out `<script>`, but allow variations like `<^script>` or `<scr#ipt>`, as they are interpreted as malformed and non-malicious.

On Firefox, comments can be closed with `--!>`, while in some markdown parsers only `-->` is recognized to close comments, allowing you to escape comments and execute XSS. These quirks present challenges for both filters and purifiers, as they are often unable to account for the wide variety of edge cases across different browsers.

![Image of XSS comment bypass](https://files.catbox.moe/633904.png)

This post will detail the website I've created for automatically fuzzing these browser parsing edge-cases along with any interesting findings across the way.

### Real World - Hacking SpaceHey.com
[SpaceHey.com](https://spacehey.com/home) is a "retro social network focused on privacy and customizability", based on the original layout and feeling of MySpace. User profiles allow you to embed basic HTML tags in your about section, which are parsed on the server-side and displayed to your profile. Here is an example of a customized user-profile with custom `<style>` tags:
![Image of example SpaceHey account](https://files.catbox.moe/2yfr4o.png)

Setting my about me section to the following contents. You can see how this is displayed and parsed by the server.
{% highlight html %}
<h1>Hello world</h1>
<script>alert(1)</script>
<!-- hello world -->
<!-- <script> -->
{% endhighlight %}

![Image of About section contents HTML](https://files.catbox.moe/wiqhil.png)
![Image of About section contents](https://files.catbox.moe/05k3on.png)

From this alone, there is quite a few notes to take on how the server handled this request:
1. The `h1` tag is rendered to the DOM without being modified.
2. The `<script>` tag was removed from the string, only leaving "alert(1)".
3. **Anything inside a comment tag is not filtered**.

It can be assumed that the filter algorithm is *basic* and is checking hardcoded values like `"-->"` to determine if an HTML comment has ended. Using the trick from above for closing comments with `--!>`, you can trick the server-side parsing into thinking the comment has not closed. Use this example of an "About me" section input:
{% highlight html %}
<!--  --!>
<script>alert(1)</script> --> 
{% endhighlight %}

![Image of About section contents HTML](https://files.catbox.moe/7v4io7.png)
![Image of About section contents](https://files.catbox.moe/g4fupx.png)

Even the Jekyll HTML highlighter for this writeup does not recognize that the comment has ended after `--!>`. As you can see, SpaceHey ignored the closing `--!>` as the end of the comment, and wrote the script tag to the DOM. Leading to stored XSS on SpaceHey profiles.

This vulnerability has been responsibly disclosed and patched by the developer of SpaceHey a few years ago now. The real MySpace was hacked using the same vulnerability, XSS. Here is a very interesting video detailing that incident and the impacts this could have on the website:

<iframe width="100%" height="500px" src="https://www.youtube.com/embed/DtnuaHl378M" title="Greatest Moments in Hacking History: Samy Kamkar Takes Down Myspace" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

### Fuzzer
Here is a screenshot of the website in its current state. It takes a "Test String" parameter which is the vector you wish to fuzz. `{payload}` and `{unicode}` are placeholder values which are replaced at runtime.

- **`{payload}`** - The JavaScript which is used to determine if XSS was executed.
- **`{unicode}`** - The placeholder which is replaced with every possible unicode character.


![Image of XSS Fuzzer](https://files.catbox.moe/x4dzjk.png)

Using a similar payload to the SpaceHey example, I am checking if any character inbetween the `--` and `>` of an HTML comment tag can be treated as a closing comment tag. Automating this could help find unique XSS vectors and exploit niche parsers.

In this case, it was able to discover that you can use the following symbols:
- `>`
- `-`
- `!`

The first two can be ignored as those are expected to close the comment tag, but the exclamation point symbol is unique and was discovered within seconds of running this tool.

### [Link to Repository](https://github.com/dceit/browser-fuzzer)
### [Link to Website](https://dceit.github.io/browser-fuzzer/fuzzer.html)