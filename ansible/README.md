
# General Connectivity Notes

The playbooks assume that it's possible to connect to each node using 'ssh'.

What this means is that you need to configure your '~/.sshconfig' to allow
access to each node.  In some cases you may require a slightly different setup
when first configuring the node, vs after the first configuration.  This will
happen when the node is remotely accessible.  

## Raspberry Pi nodes for Remote Sites

Typically these will be setup initially on a local area network, and then
maintained using remote access through the Ovpn.  
As such, the initial entry in "~/.ssh/config" will look something
like:

    # Access locally
    Host barcpi004
    HostName 192.168.0.10
    user pi

Then, when the node is installed remotely, and we want to go through the gateway,
we connect our local machine to the OVPN gateway and use the node's OVPN IP:

    # For remote access
    Host barcpi004
    HostName 10.75.0.201
    user pi

If you're an authorized administrator for the BARC infrastructure, your public
key will be in the set of public keys that are uploaded to the
remotely-accessible machines, and you should be good to go.  If you need to
use a different key file, or "identity" for "ssh", you can add the "Identity"
attribute to the config file, e.g. for initial setup of the gateway using the
key generated by EC2:

    Host barc-gateway
    Hostname gateway.barc.ca
    IdentityFile ~/certs/barc-gateway-key.pem
    user ec2-user

# Setting up a new Raspberry Pi Node

The following instructions describe the steps to setting up a new Raspberry Pi
node for the digipeater.  This can be done on an SD card image, which can then
be installed at the repeater shack.

1. Download and burn an image of
[Raspbian Jessie Lite] (https://www.raspberrypi.org/downloads/raspbian/)
according to the instructions on that page.  

2. Boot the pi using the new image.  

3. Use 'sudo raspi-config' to set the following items:  
  ssh: on  
  hostname: as required (for the rest of this doc, we'll call it 'barcpi003')

4. Reboot the pi, so the new hostname is activated.  Make sure the pi is on the
local network.  

5. Use 'ifconfig' to find out the pi's IP address.

6. Make an entry into the local '/etc/hosts' for the hostname and ip address, or
make an entry in "~/.ssh/config" as described above.

7. Test the ssh connection.

7. On the new pi, create a new ssh key

    ssh-keygen -b 2048 -t rsa

8. ssh into the pi and copy your public key to its 'authorized_keys' file under
the 'pi' user, as follows:

    cat ~/.ssh/id_rsa.pub | ssh pi@barcpi003 "cat >>~/.ssh/authorized_keys"  

8. Copy the pi's public key into the authorized_keys folder in Ansible.

    ssh pi@barcpi004 "cat ~/.ssh/id_rsa.pub" > authorized_keys/pi@barcpi003_rsa.pub

8. Add the new host to the 'hosts' file and also add a 'host_vars' file for the
new host.

9. Setup the OpenVPN client:

    ansible-playbook -i hosts -l barcpi004 setup-openvpn.yml

10. If the new node needs aprx, run the 'install-aprx' playbook.

    ansible-playbook -i hosts -l barcpi004 install-aprx.yml

11. Run the site playbook to configure the new node for the rest of the infrastructure.

    ansible-playbook -i hosts -l barcpi004 site.yml


# Setting up the EC2 Gateway

# Openvpn

The BARC infrastructure uses openvpn to allow the remote systems to have a
persistent presence on the open internet that is served through the gateway.barc.ca
domain name.  For instance, the All-Star gateways etc, need a routable address
to enable call-ins.  Also, the hope is that administrative connections like ssh may
be more efficient using the openvpn protocol than tunneling ssh connections.
There is also the possibility of having the remote machines use server
infrastructure.  For instance, we could log metrics to a database on the EC2 network.

On the openvpn infrastucture:

- We use a routed configuration  
- The subnet for connected openvpn nodes is 10.75.0.0/24 (apropos of nothing,
  just to have no conflicts).  This allows for 255 remote nodes.  
- The CA for openvpn resides on the gateway machine.  
- The gateway.barc.ca node will have ip address 10.75.0.1.  
- Any other
- Machines at the actual remote sites are assigned fixed ip addresses in
  the 10.75.0.0 subnet in the range 10.75.0.2 - 10.75.57.99  
- Temporary clients connected for admin purposes will be assigned addresses from
  the rest of the subnet.  
- Each node and the gateway needs a private key and certificate, signed by the CA   
    - The key is stored in the ansible_user's  ~/{{hostname}}.pem
    - The certificate is signed by the ca and stored in ~/{{hostname}}.crt  
- The gateway machine has a manually-installed instance of easy_rsa and the
  keys and certs are generated and distributed to the client machines manually.
