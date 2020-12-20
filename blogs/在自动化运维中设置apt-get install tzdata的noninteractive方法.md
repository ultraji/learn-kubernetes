# 在自动化运维中设置apt-get install tzdata的noninteractive方法

在Ubuntu系统中，执行命令`apt-get install -y tzdata`以安装tzdata软件包。

但是，在Ubuntu 18.04 (Bionic Beaver)上无法自动安装该软件包，这里的tzdata版本为2018d-1。在tzdata 2017的各个版本中（如2017c），安装过程中采用默认的系统时区，所以可以无交互地顺利安装完毕，输出信息如下。

    ```txt
    Current default time zone: 'Etc/UTC'
    Local time is now: Wed Apr 25 03:38:23 UTC 2018.
    Universal Time is now: Wed Apr 25 03:38:23 UTC 2018.
    Run 'dpkg-reconfigure tzdata' if you wish to change it.
    ```

但是从tzdata 2018版本开始（如2018d），安装过程中默认采用交互式，即要求输入指定的**Geographic area**和**Time zone**，从而必须人工值守进行安装，输出信息如下。

    ```txt
    Configuring tzdata
    ------------------
    Please select the geographic area in which you live. Subsequent configuration
    questions will narrow this down by presenting a list of cities, representing
    the time zones in which they are located.
    1. Africa      4. Australia  7. Atlantic  10. Pacific  13. Etc
      2. America     5. Arctic     8. Europe    11. SystemV
      3. Antarctica  6. Asia       9. Indian    12. US
    ```

这显然对于无人值守的自动化运维造成不必要的挑战。

解决步骤如下：

1. 设置tzdata的前端类型（通过环境变量）
    ```shell
    export DEBIAN_FRONTEND=noninteractive
    ```
    tzdata的前端类型默认为readline（Shell情况下）或dialog（支持GUI的情况下）。

2. 安装tzdata软件包

    ```shell
    apt-get install -y tzdata
    ```

    此时，采用默认时区Etc/UTC。

3. 建立到期望的时区的链接，设置时区为Asia/Shanghai。
    ```shell
    ln -fs /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    ```
    
4. 重新配置tzdata软件包，使得时区设置生效
    ```shell
    dpkg-reconfigure -f noninteractive tzdata
    ```