#### Secure LDAP enabled single node HDP cluster using Kerberos


- Install kerberos (On the server cloudera manager or Ambari)
```
yum -y install krb5-server krb5-libs krb5-auth-dialog krb5-workstation

- Install kerberos (On all others nodes)
```
yum -y install krb5-workstation
yum -y install krb5-libs

Can be done through Ansible : 
yum install -y epel-release
yum install -y ansible
mkdir ansible-playbook
cd ansible-playbook/
[root@node-1 ansible-playbook]# cat inventory/homelab 
[all]
node-1.testing
node-3.testing
node-4.testing
node-5.testing
node-6.testing
node-7.testing
node-8.testing
ansible -i inventory/homelab -m shell -a "yum install -y krb5-workstation krb5-libs"



- Install ldap clients on the cloudera manager server 

yum -y openldap-clients
```

- Configure kerberos
```
vi /var/lib/ambari-server/resources/scripts/krb5.conf
#edit default realm and realm info as below
default_realm = HOMELAB.COM

[realms]
 HOMELAB.COM = {
  kdc = sandbox.homelab.com
  admin_server = sandbox.homelab.com
 }

[domain_realm]
 .homelab.com = HOMELAB.COM
 homelab.com = HOMELAB.COM
```

- Copy conf file to /etc
```
cp /var/lib/ambari-server/resources/scripts/krb5.conf /etc
```
- Copy the /etc/krb5.conf file to all nodes 
```
ansible all -i inventory/homelab -m copy -a "src=/etc/krb5.conf dest=/etc/krb5.conf"

- Create kerberos db: when asked, enter hadoop as the key
```
kdb5_util create -s
```

- Start kerberos
```
/etc/rc.d/init.d/krb5kdc start
/etc/rc.d/init.d/kadmin start

chkconfig krb5kdc on
chkconfig kadmin on
```
- Create admin principal
```
kadmin.local -q "addprinc admin/admin"
kadmin.local -q "addprinc -pw hadoop cloudera-scm/admin@HOMELAB.COM"
```

- Update kadm5.acl to make admin an administrator
```
vi /var/kerberos/krb5kdc/kadm5.acl
*/admin@homelab.COM *
```
- Update kdc.conf to make admin an administrator
```
vi /var/kerberos/krb5kdc/kdc.conf
max_life = 1d  
max_renewable_life = 7d
kdc_tcp_ports = 88
```
- Restart kerberos services
```
/etc/rc.d/init.d/krb5kdc restart
/etc/rc.d/init.d/kadmin restart
```

- Login to Ambari (if server is not started, execute /root/start_ambari.sh) by opening http://sandbox.homelab.com:8080 and then
  - Admin -> Security-> click “Enable Security”
  - On "get started” page, click Next
  - On “Configure Services”, click Next to accept defaults
  - On “Create Principals and Keytabs”, click “Download CSV”. Save to sandbox by “vi /root/host-principal-keytab-list.csv" and pasting the content
  - Without pressing “Apply", go back to terminal 

- add below line to  to csv file 
```
vi host-principal-keytab-list.csv
sandbox.homelab.com,Hue,hue/sandbox.homelab.com@HOMELAB.COM,hue.service.keytab,/etc/security/keytabs,hue,hadoop,400
```

- Execute below to generate principals, key tabs 
```
/var/lib/ambari-server/resources/scripts/kerberos-setup.sh /root/host-principal-keytab-list.csv ~/.ssh/id_rsa
```

- verify keytabs and principals got created (should return at least 17)

if rm.service.keytab was not created re-run with a csv that just contains the line containing rm, otherwise YARN service will not come up
```
ls -la /etc/security/keytabs/*.keytab | wc -l
```

- check that keytab info can be ccessed by klist
```
klist -ekt /etc/security/keytabs/nn.service.keytab
```

- verify you can kinit as hadoop components. This should not return any errors
```
kinit -kt /etc/security/keytabs/nn.service.keytab nn/sandbox.homelab.com@HOMELAB.COM
```
- Click Apply in Ambari to enable security and restart all the components

If the wizard errors out towards the end due to a component not starting up, its not a problem: you should be able to start it up manually via Ambari

- Access HDFS as Hue user
```
su - hue
#Attempt to read HDFS: this should fail as hue user does not have kerberos ticket yet
hadoop fs -ls
#Confirm that the use does not have ticket
klist
#Create a kerberos ticket for the user
kinit -kt /etc/security/keytabs/hue.service.keytab hue/sandbox.homelab.com@HOMELAB.COM
#verify that hue user can now get ticket and can access HDFS
klist
hadoop fs -ls /user
exit
```
- This confirms that we have successfully enabled kerberos on our cluster

- Open Hue and notice it **no longer works** e.g. FileBrowser givers error
http://sandbox.homelab.com:8000

- **Make the config changes needed to make Hue work on a LDAP enbled kerborized cluster using steps [here](https://github.com/abajwa-hw/security-workshops/blob/master/Setup-Hue-kerberos-LDAP.md)**


- At this point you can not kinit as LDAP users (e.g hr1), but you may not need this if your users will use Knox for webhdfs/ODBC access
If you do need this functionality, you will need to configure OpenLDAP to talk to KDC. 

A rough guide for this can be found [here](https://github.com/abajwa-hw/security-workshops/blob/master/Setup-OpenLDAP-KDC-integration.md) 

- Extra:
On rebooting the VM you may notice that datanode service does not come up on its own and you need to start it manually via Ambari
To automate this, change startup script to start data node as root:
```
vi /usr/lib/hue/tools/start_scripts/start_deps.mf

#find the line containing 'conf start datanode' and replace with below
export HADOOP_LIBEXEC_DIR=/usr/lib/hadoop/libexec && /usr/lib/hadoop/sbin/hadoop-daemon.sh --config /etc/hadoop/conf start datanode,\
```
