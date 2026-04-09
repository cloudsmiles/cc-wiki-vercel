# shell切换为zsh

先查看本机的shell类型

```jsx
$cat /etc/shells
# List of acceptable shells for chpass(1).
# Ftpd will not allow users to connect who are not using
# one of these shells.

/bin/bash
/bin/csh
/bin/dash
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
```

如果有/bin/zsh，就说明已经安装了zsh，进行切换

```jsx
chsh -s /bin/zsh
```

reference:

[http://www.taodudu.cc/news/show-952921.html?action=onClick](http://www.taodudu.cc/news/show-952921.html?action=onClick)