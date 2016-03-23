A neat way to organize your source code
===

### Before the Start
As a developer, we deal with words, files, directories every day. You don't want your codebase messing up just like you do in your code architecture, the style, or the code itself.

In the book [How Google Works][1], it says, 'Messiness is a virtue'(page 38). It's not something around us neatly but the way we do something neatly matters. If you have not a clear way to organize, maybe you've come to the right place.

The way below is inspired from [the GOPATH Setting of Golang][2].

### Put all your code in one single place
I prefer ``~/src``. It's clear and points the user home directory thus you have all the **rwx** access.

### Classify your project
Assuming you have 4 types of projects, ``{}`` is a placeholder.

1. Company project: ``~/src/{example.com}/{project}`` 
2. Community project: ``~/src/{example.org}/{project}`` or ``~/src/github.com/{organization}/{project}``
3. Personal project: ``~/src/github.com/{you}/{project}``
4. Testing scratch project: ``~/src/scratch/{project}``

Note: I strongly recommend you to choose **github.com(or alternatives)** as root path of personal project. I know the feeling when you write something cool and want share with others. This is called social coding and what makes a good software today. However, if you don't want to open source, just keep it there and no publish proceeding.

### What if I clone a project form a remote repository?
Same, suppose the author or the community called Bob, puts it into ``~/src/github.com/Bob/{project}``.

### Bonus
After you build a project from ``~/src`` and outputs executeable file(s), resident it to ``~/bin`` if you have no other good reasons. The same goes for the ``var, usr, pkg, lib, etc.``. Be a good citizen in *Nix world.

If you write Go code, set ``GOPATH`` to ``~/src``, all your build artifacts would be seen in ``~/bin``, it's fine, isn't it?

Finally, there are no rules from this article but some personal suggestions. Thanks for reading, happy hacking!

### EOF
```json
{
  "tags": ["Programming", "Thoughts"],
  "reserved": false,
  "date": "2016-02-05T20:27:21+08:00",
  "weather": "very cold so no code",
  "summary": "",
  "location": "Guilin",
  "background": "/assets/images/sea.jpg"
}
```

[1]: http://www.amazon.com/How-Google-Works-Eric-Schmidt/dp/1455582344
[2]: https://golang.org/doc/code.html#Organization
