# 系统优化脚本支持Ubuntu和CentOS



安装系统后经常有一些基础的系统优化安装，本人比较懒，写了一个脚本，可以后期加入其他优化方面。

仓库地址：https://github.com/cby-chen/OS

后续可能会继续更新脚本

shell脚本如下

```shell
#!/bin/bash
os=$(cat /etc/os-release 2>/dev/null | grep ^ID= | awk -F= '{print $2}')

function selinuxset(){
    selinux_status=$(grep -c "SELINUX=disabled" /etc/sysconfig/selinux)
    echo "========================禁用SELINUX========================"
 if [ "$selinux_status" -eq 0 ];then
    sed  -i "s#SELINUX=enforcing#SELINUX=disabled#g" /etc/sysconfig/selinux
    setenforce 0
    grep SELINUX=disabled /etc/sysconfig/selinux
    getenforce
 else
    echo 'SELINUX已处于关闭状态'
    grep SELINUX=disabled /etc/sysconfig/selinux
    getenforce
 fi
    echo "完成禁用SELINUX"
    echo "==========================================================="
    sleep 3
}

function firewalldset(){
    echo "========================关闭firewalld======================="
    echo '关闭防火墙'
    systemctl  disable --now firewalld
    echo '验证如下'
    systemctl list-unit-files | grep firewalld
    echo '生产环境下建议启用'
    echo "==========================================================="
    sleep 3
}

function ufwset(){
    echo "========================关闭ufw============================"
    echo '关闭防火墙'
    systemctl  disable --now ufw
    echo '验证如下'
    systemctl list-unit-files | grep ufw
    echo '生产环境下建议启用'
    echo "==========================================================="
    sleep 3
}

function limitsset(){
    echo "======================修改文件描述符========================"
    echo '加大系统文件描述符最大值'
    {
    echo '* soft nofile 65536'
    echo '* hard nofile 65536'
    echo '* soft nproc 65536'
    echo '* hard nproc 65536'
    } >> /etc/security/limits.conf
    echo '查看配置内容'
    cat /etc/security/limits.conf
    echo '设置软硬资源限制'
    ulimit -Sn ; ulimit -Hn
    echo "==========================================================="
    sleep 3
}

function yumset(){
    echo "======================开始修改YUM源========================"
    echo '开始修改YUM源'
    sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' -e 's|^#baseurl=http://mirror.centos.org|baseurl=https://mirrors.tuna.tsinghua.edu.cn|g' -i.bak /etc/yum.repos.d/CentOS-*.repo
    echo '开始安装常规软件'
    yum update -y
    yum install curl git wget ntpdate lsof net-tools telnet vim lrzsz tree nmap nc sysstat epel* -y
    echo "==========================================================="
    sleep 3
}

function aptset(){
    echo "======================开始修改APT源========================"
    echo '开始修改APT源'
    apt_stat=$(cat /etc/apt/sources.list | grep -v ^\# | awk -F/ '{print $3}' | grep -v ^$  | awk 'NR==1{print}')
    sudo sed -i "s/$apt_stat/mirrors.ustc.edu.cn/g" /etc/apt/sources.list
    echo '开始安装常规软件'
    apt update -y
    apt upgrade -y
    apt install vim htop net-tools lrzsz nmap telnet ntpdate sysstat curl git wget -y
    echo "==========================================================="
    sleep 3
}

function restartset(){
    echo "===================禁用ctrl+alt+del重启===================="
    rm -rf /usr/lib/systemd/system/ctrl-alt-del.target
    echo "完成禁用ctrl+alt+del重启"
    echo "==========================================================="
    sleep 3
}

function historyset(){
    echo "========================history优化========================"
    chk_his=$(cat /etc/profile | grep HISTTIMEFORMAT |wc -l)
    if [ $chk_his -eq 0 ];then
    cat >> /etc/profile <<'EOF'
#设置history格式
export HISTTIMEFORMAT="[%Y-%m-%d %H:%M:%S] [`whoami`] [`who am i|awk '{print $NF}'|sed -r 's#[()]##g'`]: "
#记录shell执行的每一条命令
export PROMPT_COMMAND='\
if [ -z "$OLD_PWD" ];then
    export OLD_PWD=$PWD;
fi;
if [ ! -z "$LAST_CMD" ] && [ "$(history 1)" != "$LAST_CMD" ]; then
    logger -t `whoami`_shell_dir "[$OLD_PWD]$(history 1)";
fi;
export LAST_CMD="$(history 1)";
export OLD_PWD=$PWD;'
EOF
    source /etc/profile
    else
    echo "优化项已存在。"
    fi
    echo "完成history优化" 
    echo "==========================================================="
    sleep 3
}

function helloset(){
    echo "========================欢迎界面优化========================"
    cat << EOF > /etc/profile.d/login-info.sh
#!/bin/sh
#
# @Time    : 2022-04-21
# @Author  : chenby
# @Desc    : ssh login banner
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
shopt -q login_shell && : || return 0
echo -e "\033[0;32m

 #    #  ######  #       #        ####
 #    #  #       #       #       #    #
 ######  #####   #       #       #    #
 #    #  #       #       #       #    #
 #    #  #       #       #       #    #
 #    #  ######  ######  ######   ####        by chenby\033[0m"
# os
upSeconds="\$(cut -d. -f1 /proc/uptime)"
secs=\$((\${upSeconds}%60))
mins=\$((\${upSeconds}/60%60))
hours=\$((\${upSeconds}/3600%24))
days=\$((\${upSeconds}/86400))
UPTIME_INFO=\$(printf "%d days, %02dh %02dm %02ds" "\$days" "\$hours" "\$mins" "\$secs")
if [ -f /etc/redhat-release ] ; then
    PRETTY_NAME=\$(< /etc/redhat-release)
elif [ -f /etc/debian_version ]; then
   DIST_VER=\$(</etc/debian_version)
   PRETTY_NAME="\$(grep PRETTY_NAME /etc/os-release | sed -e 's/PRETTY_NAME=//g' -e  's/"//g') (\$DIST_VER)"
else
    PRETTY_NAME=\$(cat /etc/*-release | grep "PRETTY_NAME" | sed -e 's/PRETTY_NAME=//g' -e 's/"//g')
fi
if [[ -d "/system/app/" && -d "/system/priv-app" ]]; then
    model="\$(getprop ro.product.brand) \$(getprop ro.product.model)"
elif [[ -f /sys/devices/virtual/dmi/id/product_name ||
        -f /sys/devices/virtual/dmi/id/product_version ]]; then
    model="\$(< /sys/devices/virtual/dmi/id/product_name)"
    model+=" \$(< /sys/devices/virtual/dmi/id/product_version)"
elif [[ -f /sys/firmware/devicetree/base/model ]]; then
    model="\$(< /sys/firmware/devicetree/base/model)"
elif [[ -f /tmp/sysinfo/model ]]; then
    model="\$(< /tmp/sysinfo/model)"
fi
MODEL_INFO=\${model}
KERNEL=\$(uname -srmo)
USER_NUM=\$(who -u | wc -l)
RUNNING=\$(ps ax | wc -l | tr -d " ")
# disk
totaldisk=\$(df -h -x devtmpfs -x tmpfs -x debugfs -x aufs -x overlay --total 2>/dev/null | tail -1)
disktotal=\$(awk '{print \$2}' <<< "\${totaldisk}")
diskused=\$(awk '{print \$3}' <<< "\${totaldisk}")
diskusedper=\$(awk '{print \$5}' <<< "\${totaldisk}")
DISK_INFO="\033[0;33m\${diskused}\033[0m of \033[1;34m\${disktotal}\033[0m disk space used (\033[0;33m\${diskusedper}\033[0m)"
# cpu
cpu=\$(awk -F':' '/^model name/ {print \$2}' /proc/cpuinfo | uniq | sed -e 's/^[ \t]*//')
cpun=\$(grep -c '^processor' /proc/cpuinfo)
cpuc=\$(grep '^cpu cores' /proc/cpuinfo | tail -1 | awk '{print \$4}')
cpup=\$(grep '^physical id' /proc/cpuinfo | wc -l)
CPU_INFO="\${cpu} \${cpup}P \${cpuc}C \${cpun}L"
# get the load averages
read one five fifteen rest < /proc/loadavg
LOADAVG_INFO="\033[0;33m\${one}\033[0m / \${five} / \${fifteen} with \033[1;34m\$(( cpun*cpuc ))\033[0m core(s) at \033[1;34m\$(grep '^cpu MHz' /proc/cpuinfo | tail -1 | awk '{print \$4}')\033 MHz"
# mem
MEM_INFO="\$(cat /proc/meminfo | awk '/MemTotal:/{total=\$2/1024/1024;next} /MemAvailable:/{use=total-\$2/1024/1024; printf("\033[0;33m%.2fGiB\033[0m of \033[1;34m%.2fGiB\033[0m RAM used (\033[0;33m%.2f%%\033[0m)",use,total,(use/total)*100);}')"
# network
# extranet_ip=" and \$(curl -s ip.cip.cc)"
IP_INFO="\$(ip a | grep glo | awk '{print \$2}' | head -1 | cut -f1 -d/)\${extranet_ip:-}"
# Container info
CONTAINER_INFO="\$(sudo /usr/bin/crictl ps -a -o yaml 2> /dev/null | awk '/^  state: /{gsub("CONTAINER_", "", \$NF) ++S[\$NF]}END{for(m in S) printf "%s%s:%s ",substr(m,1,1),tolower(substr(m,2)),S[m]}')Images:\$(sudo /usr/bin/crictl images -q 2> /dev/null | wc -l)"
# info
echo -e "
 Information as of: \033[1;34m\$(date +"%Y-%m-%d %T")\033[0m
 
 \033[0;1;31mProduct\033[0m............: \${MODEL_INFO}
 \033[0;1;31mOS\033[0m.................: \${PRETTY_NAME}
 \033[0;1;31mKernel\033[0m.............: \${KERNEL}
 \033[0;1;31mCPU\033[0m................: \${CPU_INFO}
 \033[0;1;31mHostname\033[0m...........: \033[1;34m\$(hostname)\033[0m
 \033[0;1;31mIP Addresses\033[0m.......: \033[1;34m\${IP_INFO}\033[0m
 \033[0;1;31mUptime\033[0m.............: \033[0;33m\${UPTIME_INFO}\033[0m
 \033[0;1;31mMemory\033[0m.............: \${MEM_INFO}
 \033[0;1;31mLoad Averages\033[0m......: \${LOADAVG_INFO}
 \033[0;1;31mDisk Usage\033[0m.........: \${DISK_INFO} 
 \033[0;1;31mUsers online\033[0m.......: \033[1;34m\${USER_NUM}\033[0m
 \033[0;1;31mRunning Processes\033[0m..: \033[1;34m\${RUNNING}\033[0m
 \033[0;1;31mContainer Info\033[0m.....: \${CONTAINER_INFO}
"
EOF
    echo "==========================================================="
    sleep 3
}

function sshset(){
    echo "========================root登录优化========================"
    echo "生产环境不建议开启 设置root密码"
    read -p "输入root密码" rootpw
    echo "root:$rootpw" |chpasswd
    echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
    systemctl restart sshd
    echo "root密码修改为$rootpw"
    echo "==========================================================="
    sleep 3
}


function allin() {
    if [ "$os" = "\"centos\"" ]; then
        selinuxset
        firewalldset
        limitsset
        yumset
        restartset
        historyset
        helloset
    fi
    if [ "$os" = "ubuntu" ]; then
        sshset
        ufwset
        limitsset
        aptset
        restartset
        historyset
        helloset
    fi
}

function menu() {
    clear
    echo "#####################################################################"
    echo -e "#           ${RED}一键基础优化脚本${PLAIN}                        #"
    echo -e "# ${GREEN}作者${PLAIN}: chenby                                   #"
    echo -e "# ${GREEN}网址${PLAIN}: https://www.oiox.cn                      #"
    echo -e "# ${GREEN}版本${PLAIN}: V1.0                                     #"
    echo -e "# ${GREEN}说明${PLAIN}:                                          #"
    echo -e "#                                                               #"
    echo "####################################################################"
    echo " -------------"
    echo -e "  ${GREEN}1.${PLAIN}  一键优化"
    echo " -------------"
    echo -e "  ${GREEN}2.${PLAIN}  自定义优化"
    echo " -------------"
    echo -e "  ${GREEN}0.${PLAIN}   退出"
    echo " -------------"

    read -p " 请选择操作[0-2]：" chenby
    case $chenby in
        0)
            exit 0
            ;;
        1)
            allin
            ;;
        2)
            setun
            ;;
        *)
            colorEcho $RED " 请选择正确的操作！"
            exit 1
            ;;
    esac
}

function setun() {
    echo " -------------"
    echo -e "  ${GREEN}1.${PLAIN}  禁用SELINUX"
    echo " -------------"
    echo -e "  ${GREEN}2.${PLAIN}  关闭firewalld"
    echo " -------------"
    echo -e "  ${GREEN}3.${PLAIN}  关闭ufw"
    echo " -------------"
    echo -e "  ${GREEN}4.${PLAIN}  修改文件描述符"
    echo " -------------"
    echo -e "  ${GREEN}5.${PLAIN}  开始修改YUM源"
    echo " -------------"
    echo -e "  ${GREEN}6.${PLAIN}  开始修改APT源"
    echo " -------------"
    echo -e "  ${GREEN}7.${PLAIN}  禁用ctrl+alt+del重启"
    echo " -------------"
    echo -e "  ${GREEN}8.${PLAIN}  history优化"
    echo " -------------"
    echo -e "  ${GREEN}9.${PLAIN}  欢迎界面优化"
    echo " -------------"
    echo -e "  ${GREEN}10.${PLAIN} 设置root密码"
    echo " -------------"    
    echo -e "  ${GREEN}0.${PLAIN}  退出"
    echo " -------------"
    
    read -p " 请选择操作[0-2]：" cby
    case $cby in
        0)
            exit 0
            ;;
        1)
            if [ "$os" = "\"centos\"" ]; then
                selinuxset
            fi
            if [ "$os" = "ubuntu" ]; then
                echo 'Ubuntu无需设置'
            fi
            ;;
        2)
            if [ "$os" = "\"centos\"" ]; then
                firewalldset
            fi
            if [ "$os" = "ubuntu" ]; then
                echo 'Ubuntu无需设置'
            fi
            ;;
        3)
            if [ "$os" = "\"centos\"" ]; then
                echo 'CentOS无需设置'
            fi
            if [ "$os" = "ubuntu" ]; then
                ufwset
            fi
            ;;
        4)
            limitsset
            ;;
        5)
            if [ "$os" = "\"centos\"" ]; then
                yumset
            fi
            if [ "$os" = "ubuntu" ]; then
                echo 'Ubuntu无需设置'
            fi
            ;;
        6)
            if [ "$os" = "\"centos\"" ]; then
                echo 'CentOS无需设置'
            fi
            if [ "$os" = "ubuntu" ]; then
                aptset
            fi
            ;;
        7)
            restartset
            ;;
        8)
            historyset
            ;;
        9)
            helloset
            ;;
        10)
            if [ "$os" = "\"centos\"" ]; then
                echo 'CentOS无需设置'
            fi
            if [ "$os" = "ubuntu" ]; then
                sshset
            fi
            ;;
        *)
            colorEcho $RED " 请选择正确的操作！"
            exit 1
            ;;
    esac
}


if [ $(id -u) -eq 0 ];then
	menu
else
	echo "非root用户!请使用root用户！！！"
    exit 1
fi

```


# 技术交流

作者:  

![avatar](https://www.oiox.cn/about/2.png)  

加群:  

![avatar](https://www.oiox.cn/about/1.png)  

</br>
</br>

> **关于**
>
> https://www.oiox.cn/
>
> https://www.oiox.cn/index.php/start-page.html
>
> **CSDN、GitHub、知乎、开源中国、思否、掘金、简书、华为云、阿里云、腾讯云、哔哩哔哩、今日头条、新浪微博、个人博客**
>
> **全网可搜《小陈运维》**
>
> **文章主要发布于微信公众号**
