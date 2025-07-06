# 附录 D. 持久化 Solaris 网络调优

在第八章，*故障排除技术*中，我们看到了如何为不同的操作系统更改不同的网络调优参数。此附录详细说明了在 Solaris 10 及以上版本中使这些更改持久化所需的步骤。

以下脚本是**服务管理框架**（**SMF**）实际运行的内容，用于通过 ndd 设置网络参数。将其保存为`/lib/svc/method/network-tuning.sh`并使其具有可执行权限，这样就可以随时在命令行运行以进行测试：

```
# vi /lib/svc/method/network-tuning.sh

```

以下代码片段是`/lib/svc/method/network-tuning.sh`文件的内容：

```
#!/sbin/sh
# Set the following values as desired
ndd -set /dev/tcp tcp_max_buf 16777216
ndd -set /dev/tcp tcp_smallest_anon_port 1024
ndd -set /dev/tcp tcp_largest_anon_port 65535
ndd -set /dev/tcp tcp_conn_req_max_q 1024
ndd -set /dev/tcp tcp_conn_req_max_q0 4096
ndd -set /dev/tcp tcp_xmit_hiwat 1048576
ndd -set /dev/tcp tcp_recv_hiwat 1048576
# chmod 755 /lib/svc/method/network-tuning.sh

```

以下清单用于定义网络调优服务，并将在启动时运行该脚本。请注意，我们指定了*临时*的持续时间，以便 SMF 知道这是一个只运行一次的脚本，而不是一个持久的守护进程。

将其放置在`/var/svc/manifest/site/network-tuning.xml`中，并使用以下命令导入：

```
# svccfg import /var/svc/manifest/site/network-tuning.xml

```

您应该看到以下输出：

```
<?xml version="1.0"?>
<!DOCTYPE service_bundle SYSTEM "/usr/share/lib/xml/dtd/service_bundle.dtd.1">

<service_bundle type='manifest' name='SUNW:network_tuning'>

<service
       name='site/network_tuning'
       type='service'
       version='1'>

       <create_default_instance enabled='true' />

       <single_instance />

       <dependency
               name='usr'
               type='service'
               grouping='require_all'
               restart_on='none'>
               <service_fmri value='svc:/system/filesystem/minimal' />
       </dependency>

<!-- Run ndd commands after network/physical is plumbed. -->
       <dependency
               name='network-physical'
               grouping='require_all'
               restart_on='none'
               type='service'>
               <service_fmri value='svc:/network/physical' />
       </dependency>

<!-- but run the commands before network/initial -->
       <dependent
               name='ndd_network-initial'
               grouping='optional_all'
               restart_on='none'>
               <service_fmri value='svc:/network/initial' />
       </dependent>

       <exec_method
               type='method'
               name='start'
               exec='/lib/svc/method/network-tuning.sh'
               timeout_seconds='60' />

       <exec_method
               type='method'
               name='stop'
               exec=':true'
               timeout_seconds='60' />

       <property_group name='startd' type='framework'>
               <propval name='duration' type='astring' value='transient' />
       </property_group>

       <stability value='Unstable' />

       <template>
               <common_name>
                       <loctext xml:lang='C'>
                               Network Tunings
                       </loctext>
               </common_name>

       </template>
</service>

</service_bundle>
```

此服务故意保持简单，旨在演示。感兴趣的读者可以通过 Solaris 手册页和在线资源进一步探索 SMF。
