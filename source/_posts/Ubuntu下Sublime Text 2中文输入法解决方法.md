---
title: Ubuntu下Sublime Text 2中文输入法解决方法
date: 2016-04-01 23:00:38
tags: Sublime Text
---

## 编译动态共享库

### 保存下述代码为 sublime_imfix.c 文件

```c
/*
sublime_imfix.c
Use LD_PRELOAD to interpose some function to fix sublime input method support for linux.
By Cjacker Huang <jianzhong.huang at i-soft.com.cn>
 
gcc -shared -o libsublime-imfix.so sublime_imfix.c  `pkg-config --libs --cflags gtk+-2.0` -fPIC
LD_PRELOAD=./libsublime-imfix.so sublime_text
*/
#include <gtk/gtk.h>
#include <gdk/gdkx.h>
typedef GdkSegment GdkRegionBox;
 
struct _GdkRegion
{
  long size;
  long numRects;
  GdkRegionBox *rects;
  GdkRegionBox extents;
};
 
GtkIMContext *local_context;
 
void
gdk_region_get_clipbox (const GdkRegion *region,
            GdkRectangle    *rectangle)
{
  g_return_if_fail (region != NULL);
  g_return_if_fail (rectangle != NULL);
 
  rectangle->x = region->extents.x1;
  rectangle->y = region->extents.y1;
  rectangle->width = region->extents.x2 - region->extents.x1;
  rectangle->height = region->extents.y2 - region->extents.y1;
  GdkRectangle rect;
  rect.x = rectangle->x;
  rect.y = rectangle->y;
  rect.width = 0;
  rect.height = rectangle->height;
  //The caret width is 2;
  //Maybe sometimes we will make a mistake, but for most of the time, it should be the caret.
  if(rectangle->width == 2 && GTK_IS_IM_CONTEXT(local_context)) {
        gtk_im_context_set_cursor_location(local_context, rectangle);
  }
}
 
//this is needed, for example, if you input something in file dialog and return back the edit area
//context will lost, so here we set it again.
 
static GdkFilterReturn event_filter (GdkXEvent *xevent, GdkEvent *event, gpointer im_context)
{
    XEvent *xev = (XEvent *)xevent;
    if(xev->type == KeyRelease && GTK_IS_IM_CONTEXT(im_context)) {
       GdkWindow * win = g_object_get_data(G_OBJECT(im_context),"window");
       if(GDK_IS_WINDOW(win))
         gtk_im_context_set_client_window(im_context, win);
    }
    return GDK_FILTER_CONTINUE;
}
 
void gtk_im_context_set_client_window (GtkIMContext *context,
          GdkWindow    *window)
{
  GtkIMContextClass *klass;
  g_return_if_fail (GTK_IS_IM_CONTEXT (context));
  klass = GTK_IM_CONTEXT_GET_CLASS (context);
  if (klass->set_client_window)
    klass->set_client_window (context, window);
 
  if(!GDK_IS_WINDOW (window))
    return;
  g_object_set_data(G_OBJECT(context),"window",window);
  int width = gdk_window_get_width(window);
  int height = gdk_window_get_height(window);
  if(width != 0 && height !=0) {
    gtk_im_context_focus_in(context);
    local_context = context;
  }
  gdk_window_add_filter (window, event_filter, context);
}
```

### 编译动态库

编译之前请安装编译环境和GTK，apt-get安装如下：

```powershell
# sudo apt-get install build-essential
# sudo apt-get install libgtk2.0-dev
```

编译动态库，然后将编译的文件libsublime-imfix.so拷贝到Sublime Text 2安装目录下

```powershell
# sudo gcc -shared -o libsublime-imfix.so sublime_imfix.c  `pkg-config --libs --cflags gtk+-2.0` -fPIC
# mv libsublime-imfix.so /opt/sublime
```

### 启动 Sublime Text 2

进入Sublime Text 2安装目录，执行下述命令启动 Sublime Text 2，查看是否可以使用输入中文

```powershell
# cd /opt/sublime
# LD_PRELOAD=./libsublime-imfix.so ./sublime_text
```

报错：

```powershell
ERROR: ld.so: object 'libsublime-imfix.so' from LD_PRELOAD cannot be preloaded (cannot open shared object file): ignored.
```
解决方法：
注意：路径要写完整的绝对路径，启动命令修改为

```powershell
# LD_PRELOAD=/opt/sublime/libsublime-imfix.so ./sublime_text
```

启动Sublime Text 2后，便可输入中文，但是为了不用每次输入一长串命令启动Sublime Text，自己编写一个启动脚本并将修改desktop文件

## 编写Sublime Text 2启动脚本

编写Sublime Text 2启动脚本`/usr/bin/subl`

```powershell
# vim /usr/bin/subl

#!/bin/bash

SUBLIME_HOME="/opt/sublime"
LD_LIB=$SUBLIME_HOME/libsublime-imfix.so
sh -c "LD_PRELOAD=$LD_LIB $SUBLIME_HOME/sublime_text $@"

# chmod 755 /usr/bin/subl
```


## 修改Sublime Text 2启动方式

编辑Sublime Text 2 desktop文件

```powershell
# vim /usr/share/applications/sublime.desktop

[Desktop Entry]
Name=SublimeText 2
GenericName=Text Editor
Exec=subl

Terminal=false
Icon=/opt/sublime/Icon/48x48/sublime_text.png
Type=Application
Categories=TextEditor;IDE;Development
X-Ayatana-Desktop-Shortcuts=NewWindow

[NewWindow Shortcut Group]
Name=New Window
Exec=subl -n
TargetEnvironment=Unity
```

## 参考链接

- [Ubuntu系统下Sublime Text 2中fcitx中文输入法的解决方法](https://my.oschina.net/wugaoxing/blog/121281)
- [完美解决 Linux 下 Sublime Text 中文输入](https://my.oschina.net/tsl0922/blog/113495?p=2)
- [Linux下Sublime Text 2&3中文输入解决方案](https://iwww.me/125.html)
