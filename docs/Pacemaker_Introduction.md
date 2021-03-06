### 1. Giới thiệu chung
Pacemaker là thành phần trong cluster đảm trách việc quản lý tài nguyên (resource management)

![pacemaker_relate_to_otherpart](/images/pacemaker_1.jpg)


#### 1.1. Resource Agents
Để thực hiện việc quản lý, nó sử dụng các resource agents. Resource agents là một script cluster dùng để start, stop, giám sát resources. Resource agent cũng định nghĩa các thành phần được quản lý bởi cluster.

#### 1.2. Corosync/cman
Corosync là lớp quản lý node membership, Pacemaker nhận cập nhật về các thay đổi trong trạng thái cluster membership, dựa vào đó để kích hoạt các sự kiện, vd như di dời resource.


#### 1.3. Storage layer
Pacemaker có thể được sử dụng để quản lý các thiết bị lưu trữ chia sẻ (shared storage devices). Để làm được điều này, cần phải sử dụng một thành phần quản lý lock phân tán (distributed lock manager_DLM). DLM đảm trách việc đồng bộ lock giữa các thiết bị lưu trữ, điều này đặc biệt quan trọng nếu lưu trữ chia sẻ được sử dụng, VD như cLVM2 clusterd logical volumes hoặc GSF2 và OCF2 clustered file systems.

### 2. Thành phần

![pacemaker_components](/images/pacemaker_2.jpg)

Trong Pacemaker, các thành phần giao tiếp với nhau để quyết định resource nào được chạy

#### 2.1. Cluster Information Base (CIB)

Đây là trái tim của cluster. Trạng thái này luôn chạy in-memory, liên tục đồng bộ giữa các node trong cluster. Với một cluster administrator, không cần thiết phải sửa đổi CIB, nhưng đây sẽ là nguồn thông tin quan trọng khi cần debug.

Định dạng của CIB có các trường như sau:
```
<configuration>
<crm_config>
..
</crm_config>
<nodes>
..
</nodes>
<resources>
<primitive>
..
</primitive>
</configuration>
<status>
..
</status>
```
Trong CIB có 2 phần chính. Phần 1 chứa các thông tin cấu hình hệ thống, phần sau chứa các thông tin trạng thái. Trong phần cấu hình, có 3 phần chính.

Đầu tien là `crm_config`, phần này chứa các thông số cấu hình áp dụng cho toàn bộ Pacemaker.
```
<crm_config>
      <cluster_property_set id="cib-bootstrap-options">
        <nvpair id="cib-bootstrap-options-dc-version" name="dc-version" value="1.1.10-42f2063"/>
        <nvpair id="cib-bootstrap-options-cluster-infrastructure" name="cluster-infrastructure" value="corosync"/>
        <nvpair name="no-quorum-policy" value="ignore" id="cib-bootstrap-options-no-quorum-policy"/>
        <nvpair name="stonith-enabled" value="false" id="cib-bootstrap-options-stonith-enabled"/>
      </cluster_property_set>
</crm_config>
```
Tiếp là phần `<node>`, ở đây chứa các thông tin về tất cả các node trong cluster. 
```
 <nodes>
      <node id="1" uname="controller1.test"/>
      <node id="2" uname="controller2.test"/>
      <node id="3" uname="controller3.test"/>
 </nodes>
```
Cuối cùng là phần `resources`, định nghĩa tất cả các resources được quản lý bởi cluster.
```
<resources>
      <primitive id="p_haproxy" class="ocf" provider="fuel" type="ns_haproxy">
        <operations>
          <op name="monitor" interval="30" timeout="60" id="p_haproxy-monitor-30"/>
          <op name="start" interval="0" timeout="60" id="p_haproxy-start-0"/>
          <op name="stop" interval="0" timeout="60" id="p_haproxy-stop-0"/>
        </operations>
        <instance_attributes id="p_haproxy-instance_attributes">
          <nvpair name="ns" value="haproxy" id="p_haproxy-instance_attributes-ns"/>
          <nvpair name="debug" value="true" id="p_haproxy-instance_attributes-debug"/>
          <nvpair name="other_networks" value="172.16.69.0/24" id="p_haproxy-instance_attributes-other_networks"/>
        </instance_attributes>
        <meta_attributes id="p_haproxy-meta_attributes">
          <nvpair name="migration-threshold" value="3" id="p_haproxy-meta_attributes-migration-threshold"/>
          <nvpair name="failure-timeout" value="120" id="p_haproxy-meta_attributes-failure-timeout"/>
          <nvpair name="target-role" value="Started" id="p_haproxy-meta_attributes-target-role"/>
        </meta_attributes>
      </primitive>
      <primitive id="vip__public" class="ocf" provider="fuel" type="ns_IPaddr2">
        <operations>
          <op name="monitor" interval="5" timeout="20" id="vip__public-monitor-5"/>
          <op name="start" interval="0" timeout="30" id="vip__public-start-0"/>
        </operations>
        <instance_attributes id="vip__public-instance_attributes">
          <nvpair name="bridge" value="br-ex" id="vip__public-instance_attributes-bridge"/>
          <nvpair name="base_veth" value="v_public" id="vip__public-instance_attributes-base_veth"/>
          <nvpair name="ns_veth" value="b_public" id="vip__public-instance_attributes-ns_veth"/>
          <nvpair name="ip" value="172.16.69.190" id="vip__public-instance_attributes-ip"/>
          <nvpair name="iflabel" value="ka" id="vip__public-instance_attributes-iflabel"/>
          <nvpair name="cidr_netmask" value="24" id="vip__public-instance_attributes-cidr_netmask"/>
          <nvpair name="ns" value="haproxy" id="vip__public-instance_attributes-ns"/>
          <nvpair name="gateway_metric" value="10" id="vip__public-instance_attributes-gateway_metric"/>
          <nvpair name="gateway" value="172.16.69.1" id="vip__public-instance_attributes-gateway"/>
        </instance_attributes>
        <meta_attributes id="vip__public-meta_attributes">
          <nvpair name="migration-threshold" value="3" id="vip__public-meta_attributes-migration-threshold"/>
          <nvpair name="failure-timeout" value="60" id="vip__public-meta_attributes-failure-timeout"/>
          <nvpair name="resource-stickiness" value="1" id="vip__public-meta_attributes-resource-stickiness"/>
        </meta_attributes>
      </primitive>
 </resources>
```

Phần thứ 2 của CIB là phần dài nhất. Nó chứa thông tin trạng thái hiện thời về các resource trong cluster. Nó cho biết chuyện gì đang xảy ra trong cluster, đây là các thông tin debug quan trọng cho administrator.

```
  <status>
    <node_state id="1" uname="controller1.test" in_ccm="true" crmd="online" crm-debug-origin="post_cache_update" join="membe
r" expected="member">
      <transient_attributes id="1">
        <instance_attributes id="status-1">
          <nvpair id="status-1-probe_complete" name="probe_complete" value="true"/>
        </instance_attributes>
      </transient_attributes>
      <lrm id="1">
        <lrm_resources>
          <lrm_resource id="vip__public" type="ns_IPaddr2" class="ocf" provider="fuel">
            <lrm_rsc_op id="vip__public_last_0" operation_key="vip__public_start_0" operation="start" crm-debug-origin="buil
d_active_RAs" crm_feature_set="3.0.7" transition-key="8:0:0:63427bb5-fc45-4ec4-bcff-6bda4cea328a" transition-magic="0:0;8:0:
0:63427bb5-fc45-4ec4-bcff-6bda4cea328a" call-id="13" rc-code="0" op-status="0" interval="0" last-run="1482382805" last-rc-ch
ange="1482382805" exec-time="2259" queue-time="0" op-digest="329944f45c1747f8e7b40056cddfbf36"/>
            <lrm_rsc_op id="vip__public_monitor_5000" operation_key="vip__public_monitor_5000" operation="monitor" crm-debug
-origin="build_active_RAs" crm_feature_set="3.0.7" transition-key="9:0:0:63427bb5-fc45-4ec4-bcff-6bda4cea328a" transition-ma
gic="0:0;9:0:0:63427bb5-fc45-4ec4-bcff-6bda4cea328a" call-id="18" rc-code="0" op-status="0" interval="5000" last-rc-change="
1482382807" exec-time="7341" queue-time="0" op-digest="79248110b682501e477076f60d1d590a"/>
          </lrm_resource>
          <lrm_resource id="p_haproxy" type="ns_haproxy" class="ocf" provider="fuel">
            <lrm_rsc_op id="p_haproxy_last_0" operation_key="p_haproxy_start_0" operation="start" crm-debug-origin="build_ac
tive_RAs" crm_feature_set="3.0.7" transition-key="6:0:0:63427bb5-fc45-4ec4-bcff-6bda4cea328a" transition-magic="0:0;6:0:0:63
427bb5-fc45-4ec4-bcff-6bda4cea328a" call-id="16" rc-code="0" op-status="0" interval="0" last-run="1482382807" last-rc-change
="1482382807" exec-time="10830" queue-time="0" op-digest="ac7d804ed00c9fc5f1e970abf8e5c6df" op-force-restart=" dummy " op-re
start-digest="f2317cad3d54cec5d7d7aa7d0bf35cf8"/>
            <lrm_rsc_op id="p_haproxy_monitor_30000" operation_key="p_haproxy_monitor_30000" operation="monitor" crm-debug-o
rigin="build_active_RAs" crm_feature_set="3.0.7" transition-key="7:0:0:63427bb5-fc45-4ec4-bcff-6bda4cea328a" transition-magi
c="0:0;7:0:0:63427bb5-fc45-4ec4-bcff-6bda4cea328a" call-id="22" rc-code="0" op-status="0" interval="30000" last-rc-change="1
482382818" exec-time="736" queue-time="0" op-digest="4b97ee03eebf841c64253ce90572a0e6"/>
          </lrm_resource>
        </lrm_resources>
      </lrm>
    </node_state>
    <node_state id="2" uname="controller2.test" crmd="online" join="down" crm-debug-origin="post_cache_update" in_ccm="true"
/>
    <node_state id="3" uname="controller3.test" crmd="online" join="down" crm-debug-origin="post_cache_update" in_ccm="true"
/>
  </status>
```

Có nhiều loại resource cần quản lý, bao gồm:
 - *Primitives*: là service được quản lý bởi cluster. Đây là một service đơn lẻ, được quản lý bởi sysctl, runlevels, hoặc các node không thuộc cluster.
 - *Groups*: group là tập hợp các primitives. Lợi thế khi làm việc với group là cluster sẽ khởi chạy các primitives là một phần của group, theo thứ tự được định nghĩa trước trong group. Cluster cũng giữ các primitives trong cùng 1 group, nếu một primitive lỗi, các primitives tiếp sau không thể khởi chạy.
 - *Clones*: một clone là một primitive cần được khởi chạy bởi cluster nhiều hơn 1 lần. Clones sử dụng cho các service cần phải chạy ở active/active mode, VD như clustered file systems.
 - *Master slaves*: đây là một dạng đặc biệt của clone, trong đó một số instance (ít nhất là 1) là active master, các instance còn lại là slave.

#### 2.2. CRMD

 Cluster resource manager daemon(crmd) là tiến trình quản lý trạng thái hoạt động của cluster. Chức năng chính của crmd là điều khiển các luồng thông tin giữa các thành phần của cluster, VD sắp xếp resource trên một số node. Nó cũng phụ trách thay đổi trạng thái node.
 Trên mỗi cluster, có một tiến trình crmd trên mỗi node. Một trong số chúng là master. Node mà master crmd hoạt động gọi là designated coordinator(DC). Nếu DC lỗi, cluster sẽ tự động chọn một node khác làm DC thay thế.

#### 2.3. Pengine

 CIB cung cấp mô tả cho trạng thái của cluster, và policy engine(pengine) là thành phần của cluster xử lý việc này. Nó tạo ra một danh sách các chỉ dẫn(instructions) và gửi tới `crmd`. Administrator có thể tác động pengine bằng cách định nghĩa các constraint trong cluster.

#### 2.4. Lrmd

 Local resource manager daemon (lrmd) là thành phần của cluster chạy trên mỗi node. Nếu `crmd` quyết định một resource nào cần chạy trên node nào, nó sẽ ra lệnh cho `lrmd` trên node đó khởi chạy resource đó. Trong trường hợp resource đo không làm việc, lrmd sẽ thông báo lại crmd việc khởi chạy resource thất bại, cluster sau đó có thể khởi chạy resource đó trên node khác trong cluster. LRM cũng phụ trách việc giám sát việc vận hành resource và có thể dừng (stop) resource đang chạy trên 1 node.

#### 2.5. STONISHD/FENCED

 Tiến trình stonishd(hoặc fenced trên Redhat-based cluster) nhận các instruction từ crmd về việc thay đổi trạng thái node. Nếu 1 node trong cluster không trả lời cho cluster membership layer (corosync), cluster membership layer sẽ thông báo cho crmd, và crmd cảnh báo stonishd để tắt node đó. Việc vận hành như vậy đặc biệt quan trọng cho cluster, và thậm chí nếu software cho phép định nghĩa cluster mà không cần stonishd, bạn cũng không nên làm vậy, vì nó tạo ra một Cluster không tin cậy(unreliable cluster).

### 3. Các công cụ quản lý Cluster

 Có nhiều công cụ để tương tác, thay đổi trạng thái của cluster trong CIB

#### 3.1. CRM shell

 `crm shell` cung cấp giao diện tương tác với CIB. Phiên bản mới của `crm shell` cũng cho phép cấu hình corosync và các thành phần khác của cluster, để sử dụng `crm shell` phải đảm bảo package `crmsh` đã được cài đặt.

 ```
 # crm
crm(live)# help

This is the CRM command line interface program.

Available commands:

	cib manage shadow CIBs
	resource resources management
	configure CRM cluster configuration
	node nodes management
	options user preferences
	history CRM cluster history
	site Geo-cluster support
	ra resource agents information center
	status show cluster status
	quit,bye,exit exit the program
	help show help
	end,cd,up go back one level
```

#### 3.2. Hawk

High Availability Web Konsole, cung cấp cho SUSE Linux Enterprise, OpenSUSE, Debian, Fedora. HAWK gồm 2 phần:
 - Service script: khởi chạy trên node cài đặt Hawk.
 - Hawk management interface: giao diện web để quản trị, Hawk sử dụng port 7630 (https://yourserver:7630)
 ![HAWK](/images/pacemaker_hawk_3.jpg)

#### 3.4. PCS

`pcmk`(chứa pcs package) là tập lệnh đi kèm với pacemaker stack của Redhat trong Redhat 6, tính năng tương tự `crm shell`

Tham khảo:

[1] - http://clusterlabs.org/doc/en-US/Pacemaker/1.0/html/Pacemaker_Explained/s-intro-architecture.html