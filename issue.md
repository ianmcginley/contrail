the only place I can see dns_server as a set IP address is in this section of code.
 
                # assign new dns_server_address
                # reset old dns_server_address and set
                # new dns_server_address in bitmap
                if not subnetting and not routed_vn:
                    if old_dns_addr != new_dns_addr:
                        if subnet_obj.ip_belongs(old_dns_addr):
                            if IPAddress(old_dns_addr) in subnet_obj._exclude:
                                subnet_obj._exclude.remove(IPAddress(old_dns_addr))
                            subnet_obj.ip_free(old_dns_addr)
                        if subnet_obj.ip_belongs(new_dns_addr):
                            subnet_obj._exclude.append(IPAddress(new_dns_addr))
                            subnet_obj.ip_reserve(new_dns_addr, 'dns_server')  <<<<<<<<<<<<<<<<<
                        subnet_obj.dns_server_address = IPAddress(new_dns_addr)
                # update local subnet_obj with data from notify
                subnet_obj.subscriber_tag = ipam_subnet.get('subscriber_tag')
 
 
and the reason it's only written to ZooKeeper is this:
 
    # ip address management helper routines.
    # ip_set/reset_in_use do not persist to DB just internal bitmap maintenance
    # (for e.g. for notify context)
    # ip_reserve/alloc/free persist to DB
...
    def ip_reserve(self, ipaddr, value):
        ip = IPAddress(ipaddr)
        req = int(ip)
        addr = self._db_conn.subnet_reserve_req(self._name, old_div(req,self.alloc_unit), value)
        if addr:
            return str(IPAddress(addr*self.alloc_unit))
        return None
    # end ip_reserve
The comments give it away, do not persist to DB, just internal bitmap maintenance (eg: in ZooKeeper.
[10:15 am] McGinley, Ian
now to try and trace back the calculation of new_dns_addr
[10:44 am] McGinley, Ian
I come down to, this line should be keeping you out of the if statement which sets new_dns_addr as in our case (addr_from_start = null, thus network.last - 2)
 
                if not subnetting and not routed_vn:
                    if ('dns_server_address' not in ipam_subnet) or \
                        (ipam_subnet['dns_server_address'] is None) or \
                        (ipam_subnet['dns_server_address'] == 'None') or \
                        ((old_dns_addr != 'None') and \ <<<<<<<<<<<
                        (ipam_subnet['dhcp_option_list']) is None and \ <<<<<<<<<<<
                        (ipam_subnet['enable_dhcp'] == True)): <<<<<<<<<<<
                        #we need to make sure subnet_obj has a default dns
                        network = subnet_obj._network
                        if addr_from_start:
                            new_dns_addr = str(IPAddress(network.first + 2))
                        else:
                            new_dns_addr = str(IPAddress(network.last - 2))
 
Ahhh reading through, it's ANY OF THE conditions... AND you match:
 
                        ((old_dns_addr != 'None') and \ <<<<<<<<<<<
                        (ipam_subnet['dhcp_option_list']) is None and \ <<<<<<<<<<<
                        (ipam_subnet['enable_dhcp'] == True)): <<<<<<<<<<<
old_dns_addr is calcuated from:
 
                old_dns_addr = str(subnet_obj.dns_server_address)
and subnet_obj is from:
                subnet_obj = self._subnet_objs[obj_uuid][subnet_name]
old_dns_addr = 10.173.173.194 (so true) and dhcp_option_list is null/None and enable_dhcp has string true/True
 
Which is a match, so it steps into:
 
                        #we need to make sure subnet_obj has a default dns
                        network = subnet_obj._network
                        if addr_from_start:
                            new_dns_addr = str(IPAddress(network.first + 2))
                        else:
                            new_dns_addr = str(IPAddress(network.last - 2))
addr_from_start = null, so we run the else statement of network.last (broadcast) - 2
 
Which gets us new_dns_addr of 10.173.173.199 - 2 = 10.173.173.197 ----- THIS SHOULD BE THE IP THAT IS UNAVAILABLE IN 004
 
*** From the db_manage.py checker *** 
Extra IP 10.173.173.197 (IIP dns_server) in zookeeper for vn default-domain:mnsdev:V-CW-D-MNSD-VN-CMI-004
 
Which then steps into the following code:
 
                if not subnetting and not routed_vn:
                    if old_dns_addr != new_dns_addr:
                        if subnet_obj.ip_belongs(old_dns_addr):
                            if IPAddress(old_dns_addr) in subnet_obj._exclude:
                                subnet_obj._exclude.remove(IPAddress(old_dns_addr))
                            subnet_obj.ip_free(old_dns_addr)
                        if subnet_obj.ip_belongs(new_dns_addr):
                            subnet_obj._exclude.append(IPAddress(new_dns_addr))
                            subnet_obj.ip_reserve(new_dns_addr, 'dns_server')
                        subnet_obj.dns_server_address = IPAddress(new_dns_addr)
                # update local subnet_obj with data from notify
                subnet_obj.subscriber_tag = ipam_subnet.get('subscriber_tag')
old_dns_addr = 10.173.173.194
new_dns_addr = 10.173.173.197
So first condition is TRUE
 
then does the IP address of old below to the subnet - yes
 
Then moves onto does new_dns_addr below to subnet - YES
add it to the exclude list and run ip_reserve of the 10.173.173.197!
 
exclude list starts life as :         exclude = [IPAddress(network.first), IPAddress(network.broadcast)]
 
and _exclude = exclude, and _exclude.append adds it to the list.
 
Then runs ip_reserve, adding it to zookeeper
 
    # ip address management helper routines.
    # ip_set/reset_in_use do not persist to DB just internal bitmap maintenance
    # (for e.g. for notify context)
    # ip_reserve/alloc/free persist to DB
...
    def ip_reserve(self, ipaddr, value):
        ip = IPAddress(ipaddr)
        req = int(ip)
        addr = self._db_conn.subnet_reserve_req(self._name, old_div(req,self.alloc_unit), value)
        if addr:
            return str(IPAddress(addr*self.alloc_unit))
        return None
    # end ip_reserve

========================
RESOLUTION
========================

if they add:
 
         network_ipam_refs_data_ipam_subnets_addr_from_start: true

to their template, it will then equate old_dns_addr and new_dns_addr
 
which means this following piece of code,
 
                if not subnetting and not routed_vn:
                    if old_dns_addr != new_dns_addr:
                        if subnet_obj.ip_belongs(old_dns_addr):
                            if IPAddress(old_dns_addr) in subnet_obj._exclude:
                                subnet_obj._exclude.remove(IPAddress(old_dns_addr))
                            subnet_obj.ip_free(old_dns_addr)
                        if subnet_obj.ip_belongs(new_dns_addr):
                            subnet_obj._exclude.append(IPAddress(new_dns_addr))
                            subnet_obj.ip_reserve(new_dns_addr, 'dns_server')
                        subnet_obj.dns_server_address = IPAddress(new_dns_addr)
                # update local subnet_obj with data from notify
                subnet_obj.subscriber_tag = ipam_subnet.get('subscriber_tag')
will equate
 
old_dns_addr = 10.173.173.194
new_dns_addr = 10.173.173.194 
 
as FALSE and never run the ip_reserve


source:
 
https://github.com/tungstenfabric/tf-controller/blob/6a674397821187fedfffd9586942b906d3479dbc/src/config/api-server/vnc_cfg_api_server/vnc_addr_mgmt.py#L795
