# centos update kernel



# Update Kernel


```
# 导入 Key

rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org



# 安装 Yum 源

rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm



# 更新 kernel

yum --enablerepo=elrepo-kernel install -y kernel-lt kernel-lt-devel 


# 配置 内核优先

grub2-set-default 0


```

