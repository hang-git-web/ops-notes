一：快捷键：
ctrl+L 清屏，但是在上面还是能翻回来
ctrl+c 强制停止
Tab 自动补全命令，文件名，目录名
ctrl+a 光标跳至行首
ctrl+e 光标跳至行尾
ctrl+z 暂停当前前台进程放入后台
ctrl+w 删除单词，以空格间隔
ctrl+y 快捷键删除的内容恢复

vim快捷键
vim -R  只读不修改
vim +行号 filename
vim -r 恢复机制
没有上下左右，HJKL
命令行模式的批量替换
快捷键：
光标移动到行首^，行尾$，最后一行G，第一行gg，888行888G或999gg
o当前行下方添加一行，删除当前行dd，撤销到上一步：u，取消撤销：ctrl+r 
删除当前行往后的14行：14dd，粘贴回来：p
查找关键字：命令行模式，/关键字，n：到下一个匹配关键字，N向上，n向下翻
打开行号：命令行模式输入:set nu


二：Linux安全加固-修复bug，打补丁
1：更新系统与打补丁
apt update
apt install unattended-upgrades  #安装安全补丁

2：防火墙配置
sudo ufw status  #查看防火墙状态
sudo ufw allow http   # 相当于 80/tcp
查看所有规则（带序号）：
sudo iptables -L -n --line-number
-L 列出所有规则，-n 显示IP和端口数字，--line-number

3：禁止root用户远程登录-防止密码泄漏的情况
a.修改ssh配置文件 vim etc/ssh/sshd_config
b.PermitRootLogin项
c.重启ssh服务

4：SELinux
SELinux 给程序装上紧箍咒，限制访问无语
a.查看服务状态  sestatus或者getenforce
b.通过sestatus1 和sestatus0改变开始或关闭状态
或者通过配置文件来改 vim /etc/selinux/config

5：关闭不必要的服务-节省与维护
a.先查看服务 sytemctrl list
b.关闭


