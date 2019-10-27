---
layout: post
status: publish
published: true
title: Executable Octave scripts with interpreter arguments
author: Travis Mick
date: 2014-09-27
---
I prefer to make my scripts executable, rather than invoking the interpreter explicitly every time I want to run them. Most interpreters are friendly enough to make shebang lines easy to write, but Octave isn't quite the team player...

<!-- more -->

When you invoke Octave without the `-q` option, it prints an enormous informational message before executing your code. In theory, you could just change your shebang line to:

```
#!/usr/bin/octave -q
```

But, that isn't really a good idea if you're using your script on multiple systems, because octave might live in `/bin` or `/usr/local/bin` or somewhere else... We don't know. The env program is more likely to be in a consistent location, and that's why shebang lines usually look like this:

```
#!/usr/bin/env octave
```

Unfortunately, we can't just stick a `-q` on there, because we are only allowed one argument in the shebang (which in this case, is 'octave'). But, there exists a little trick. Octave's block comments look like this:

```
#{
...
#}
```

So, I came up with this solution. We will put bash (or sh) in our shebang line, then put an Octave block comment at the beginning of the script which will contain the shell commands we need to invoke Octave with the arguments we want.

```
#!/usr/bin/env bash
#{
/usr/bin/env octave -q $0 $@
exit
#}

# Octave code goes down here
```

Bash will invoke Octave and then exit, without trying to run the Octave code below (and thus without complaining about it). Octave will ignore our bash code and jump right into the actual script. Problem solved.

