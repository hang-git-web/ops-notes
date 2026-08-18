文件系统简介-仓库管理员

文件权限
三种角色类别：文件所有者(户主)，所属成员(家庭成员)，其他用户(访客)
三种权限：读写执行

权限符号：
u,g,o,a
r,w,x
+添加权限，-移除权限，=设置权限


chmod 改变权限
chmod ?+? 文件名/文件夹名 字母法
chmod 数字 文件名/文件夹名 数字法

chown 改变所有者，所属组
chmod test 文件名 ，文件所属者改为test
chmod :test 文件名 ，文件所属组改为test
chmod root:root 文件名，文件名所属者所属组都改成root
如图images1下2.png

系统目录用途
/bin 存放基本命令，系统恢复工具
/etc 系统配置文件，用户管理文件等等
/home 个人文件
/var 日志，临时文件，数据库文件

查找文件命令
find
find / -name 1.txt     查找到会输出路径

grep 可配合find一起
如图images1下3.png

软链接，硬链接
可看notes里的stage1-05-文件系统深入.md


