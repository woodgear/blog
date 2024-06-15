---
title: "the best way to save/restore your tmux session"
datePublished: Sat Jun 15 2024 15:52:48 GMT+0000 (Coordinated Universal Time)
cuid: clxgaqspz000109jv4g6f5jr7
slug: the-best-way-to-saverestore-your-tmux-session
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1718453022897/ff596373-d539-4959-bd4c-34f882c00ed8.png
tags: tmux, how-do-i-work

---

## 效果展示

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718453222762/6f127476-7517-4be6-bd24-08c25a314798.gif align="center")

## 状态是什么

准备工作总是麻烦的，在不同的task之间切换，想重新找回上下文更是让人混乱。就像计算机一样。我们需要某种机制，能去确切的保存住当前的状态。在以后的任意时间点能够切换回来

## tmux的一些问题

对于我来说，一个任务的上下文基本分为

1. 笔记，记录了当前的进度，想法，todolist等
    
2. 终端，我一般会开一个tmux session。将日志，测试，程序启动等这个任务相关的所有东西放在同一个屏幕上
    
3. 网页。firefox+tst 可以直接保存在书签中。
    
4. ide，vscode打开的各种文件夹
    

这里我们主要关注tmux的部分。

tmux很好用。但当想用他来保存状态的话有几个问题

1. tmux自己只专注于终端分屏的部分，没有自带的保存layout，恢复layout的功能
    
2. tmux直到2.3的时候（2016年。。好像也不是很晚）的时候才支持panel级别的title
    
    1. 默认的panel title是自动变化的，当你ssh的时候，会自动变成远端的主机名。这个特性看起来很好。但是当你同时连接多个相同的节点时，很难单从主机名上回想起这个pane到底是想干什么
        
        1. 与之相对的zellij在pane重命名的部分做的就很不错。但这次我们还是主要讲tmux。。
            

第一个点。save/load layout实际上有很多tmux的衍生项目在做。

[tmuxinator](https://github.com/tmuxinator/tmuxinator) [tmuxp](https://github.com/tmux-python/tmuxp) 这类的要义在于你先定义一个配置文件（yaml etc），由这些工具帮你构造出一个tmux的session。（不过tmuxp 提供了freeze 命令 能够将当前的layout保存起来）

[tmux-resurrect](https://github.com/tmux-plugins/tmux-resurrect) 更是直接把自己做成了插件，让你能够通过快捷键，保存所有的session。

tmux-resurrect 只能同时保存和还原所有的session。不能单独的保存/还原某个session

在我们看来，以上这些工具的问题在于，他们想自己做的太全了。完全没有必要。在使用者自己提供了一些hint之后，体验就能做的更好好。

## build block

如果我们想自己做一个类似的工具的话其实很简单。直接使用命令行就已经足够了。（tmux-resurrect就是纯bash写的）

下面介绍所需要用到的命令

1. tmux list-pane和tmux list window 能够列举任意session的pane和window的状态。
    

```bash
tmux list-panes -a -F 'pane    #{session_name} #{window_index} #{window_active}        :#{window_flags}        #{pane_index}   #{@mytitle}     #{@mybooter}    #{pane_active}  #{pane_current_path}'
pane    ice 0 1        :*        0   mray-run     @ . ./scripts/v2ray.actions.sh @ mray-init-and-test    0  /home/cong/sm/lab/v2ray-core
pane    ice 0 1        :*        1   vps     @@ to-baba @@ ssh root@172.245.72.75  @@ python3 -m http.server    0  /home/cong/sm/work-park/alive-kernel
pane    ice 0 1        :*        2   local     @ ls    1  /home/cong/sm/lab/v2ray-core
pane    ice 0 1        :*        3   k-curl     @ while true;do sleep 1s;date;curl -m 1 172.245.72.75:8000;done    0  /home/cong/sm/lab/v2ray-core
pane    t-tmux 0 0        :        0            1  /home/cong/sm/work-park/alive-kernel
pane    t-tmux 1 0        :-        0            0  /home/cong/sm/work-park/alive-kernel
pane    t-tmux 1 0        :-        1            0  /home/cong/sm/work-park/alive-kernel
pane    t-tmux 1 0        :-        2   f         0  /home/cong/sm/work-park/alive-kernel
pane    t-tmux 1 0        :-        3            1  /home/cong/sm/work-park/alive-kernel
pane    t-tmux 2 1        :*        0            1  /home/cong/sm/work-park/alive-kernel

tmux list-windows -a -F 'window  #{session_name} #{window_index} :#{window_name} #{window_active}        :#{window_flags}        #{window_layout}'
window  ice 0 ::zsh 1        :*        b6f4,320x81,0,0[320x40,0,0{160x40,0,0,58,159x40,161,0,59},320x40,0,41{208x40,0,41,60,111x40,209,41,61}]
window  t-tmux 0 :python 0        :        dbd3,320x81,0,0,24
window  t-tmux 1 :测 0        :-        2b24,320x81,0,0[320x40,0,0,23,320x20,0,41,26,320x19,0,62{160x19,0,62,27,159x19,161,62,28}]
window  t-tmux 2 :f 1        :*        dbd4,320x81,0,0,25
```

2. 其中window\_layout，指的就是tmux session中某个window的所有pane的具体位置。
    

具体介绍见[what-is-tmux-layout-string-format](https://stackoverflow.com/a/71324581/5642024)

当window的pane数量一致时，我们可以直接用 `tmux select-layout -t $session_name:$win_index $window\_layout` 来恢复布局。

3. `tmux set-option -p -t '0:0.0' @mytitle "xxx"`
    

可以在panel级别设置变量。-t 后跟的是pane的index 具体格式为 `$session_name:$win_index.$pane_index`

**拥有在pane级别设置变量的能力极其的重要。这样我们能够赋予这个pane的任意想要的元数据。**

从实质上来说，pane就变成了一个我们可以操作的对象了。

4. tmux config中可以自定义panel的border format。
    

也就是定义panel 分割线上显示的内容。而这里是可以用变量的 比如

```bash
set -g pane-border-format "i:#{pane_index} t:#{@mytitle} b:#{@mybooter}"
```

我一般会在panel上显示当前的index，自定义的title，booter（如何恢复这个pane的状态）

5. \`tmux send-key -t $pane\_target 'C-c' 'enter' 'xx' 'enter'\`
    
    1. 我们可以在tmux的内部或者外部使用这个指令发送任意的命令到任意的pane
        

## 解决方案

这样的话，具体的解决方式就是

1. 在每个panel使用mytitle这个变量来设置title，这样是为了不会自动重命名等特性干扰
    
2. 在每个pane上使用booter这个变量记录如何恢复状态。
    
    1. 从具体的实现上，在恢复完layout之后，会使用tmux send-key 把boot指令重发一遍。
        
    2. 我个人认为
        
        1. 想单纯的通过遍历process的方式找一个pane的boot指令是不现实的（tmux-resurrect的方式）
            
        2. 通过定义配置文件，然后resync的方式，不够直观（tmuxp等的方式）
            
        3. 在创建panel的时候，想办法将boot指令记录下来，这种通过pane option的方式暴露，这样正好能够达到一个很好的平衡。
            

### tips and hacks

#### tmux-set-panel

对于我来讲，一个pane的状态主要有两个，名字和恢复指令。简单的封装了一个tmux-set-panel的函数，可以在更新panel元数据的同时执行初始化操作。

```bash
function tmux-set-panel() {
  local t=$1
  local title=$2
  local booter=$3
  tmux set -p -t "$t" @mytitle "$title"
  tmux-boot-pane "$t" "$booter"
}

function tmux-boot-pane() {
  local t=$1
  local booter="$2"
  tmux set -p -t "$t" @mybooter "$booter"
  local cmd=$(
    cd $SHELL_ACTIONS_BASE/scripts
    python3 tmux.py gen_tmux_send_keys "$booter"
  )
  if [[ -z $t ]]; then
    cmd="tmux send-keys $cmd"
  else
    cmd="tmux send-keys -t $t $cmd"
  fi
  echo "$cmd"
  eval "$cmd"
}
```

```python
    def gen_tmux_send_keys(self, booter: str):
        delimeter = booter.split(" ")[0]
        cmds = [f"'{x.strip()}'" for x in booter.removeprefix(
            delimeter).split(delimeter)]
        cmds.insert(0, "C-c")
        full = " 'enter' ".join(cmds)
        return f"{full} 'enter';"
```

#### tmux-load

如果想把配置load进当前的tmux session，有个尴尬的问题。执行load指令如果也是在pane中，就可能导致load指令本身恰好被这个pane的booter过程打断。。

一个hack是我们可以在popup的float window中做这件事。。。

```bash
function tmux-load() (
  tmux display-popup "zsh -c \"source ~/.zshrc; cd $PWD;pwd;python $SHELL_ACTIONS_BASE/scripts/tmux.py tmux-load $1\"" &
)
```

#### wait pane ready

有件更尴尬的事情是我们如果你配置了挺多zsh/shell config，new panel的过程是需要时间的。。。

这里一个简单实用的小hack是在初始化layout之后，booter 之前。等上几秒。。

总之，我们可以很容易的通过几行代码恢复任意的tmux session

```python
    def load(self, p: str):
        exp = Layout.from_json(Path(p).read_text())
        if not self.cli.ami_intmux():
            raise Exception("在tmux环境下执行")
        session_name = self.cli.cur_session()
        cur = Layout(session_name=self.cli.cur_session(), wins=self.cli.list_window(session_name),
                     panes=self.cli.list_pane(session_name))
        for w in exp.wins:
            cur_win_index = [x.window_index for x in cur.wins]
            if w.window_index not in cur_win_index:
                self.cli.run(
                    f"tmux new-window -d -t \"{session_name}:{w.window_index}\"", hide=True)
            self.cli.run(
                f"tmux rename-window -t \"{session_name}:{w.window_index}\"  \"{w.window_name}\"", hide=True)
        # 创建panel
        for p in exp.panes:
            cur_pane_index = [x.pane_index for x in cur.panes]
            if p.pane_index not in cur_pane_index:
                self.cli.run(
                    f"tmux split-window -t \"{session_name}:{p.window_index}\" -c \"{p.pane_current_path}\"", hide=True)
        # 设置layout
        for w in exp.wins:
            self.cli.run(
                f"tmux select-layout -t \"{session_name}:{w.window_index}\" \"{w.window_layout}\"", hide=True)
        # 设置option
        for p in exp.panes:
            t = f"{session_name}:{p.window_index}.{p.pane_index}"

            cmd = f"""tmux set-option -p -t "{t}" @mytitle "{p.mytitle}" """
            self.cli.run(cmd, hide=True)

            cmd = f"""tmux set-option -p -t "{t}" @mybooter "{p.mybooter}" """
            self.cli.run(cmd, hide=True)
        sleep(3)
        # booter
        for p in exp.panes:
            t = f"{session_name}:{p.window_index}.{p.pane_index}"
            cmd = self.gen_tmux_send_keys(p.mybooter)
            self.cli.run(f""" tmux send-keys -t "{t}" {cmd} """, hide=True)
        pass

        # focus window and pane
        for p in exp.panes:
            if p.pane_active:
                t = f"{session_name}:{p.window_index}.{p.pane_index}"
            self.cli.run(
                f"""tmux switch-client -t "{session_name}:{p.window_index}" """, hide=True)
            self.cli.run(
                f"""tmux select-pane -t "{p.pane_index}" """, hide=True)
        for w in exp.wins:
            if w.window_active:
                self.cli.run(
                    f"""tmux select-window -t "{session_name}:{w.window_index}" """, hide=True)
        pass
```

## todolist

1. 整个save的过程完全可以发生tmux session的外部。所以我们只要写一个cronjob就能做的自动的定时save了。
    
2. load也是同样。可以在创建一个tmux session，然后设置他的各种layout
    
3. 实际上要保存状态，tmux-resurrect还做了挺多。比如zoom的状态，对pane内程序进行感知。保存vim layout之类的
    

## 使用到的代码

[tmux.py](https://github.com/awesome-code-actions/awesome-shell-actions/blob/master/scripts/tmux.py)

[tmux-actions](https://github.com/awesome-code-actions/awesome-shell-actions/blob/master/scripts/tmux-actions.sh)