#ocp-installation


### KVM 기준 의존성 설치
```
sudo apt update
sudo apt install -y qemu-kvm libvirt-clients libvirt-daemon-system virtinst bridge-utils 
yum -y install qemu-kvm qemu-system libvirt libvirt-bin libvirt-client virt-install virt-viewer virt-manager
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt $USER 
```


### RedHat Assisted Installer 기준
https://console.redhat.com/openshift/clusters/list  
https://console.redhat.com/openshift/create/datacenter

Datacenter - Create cluster  

Assisted Installer를 통해 반자동화 환경을 세팅할 수 있습니다.

```
Cluser name : 클러스터 명 (Base domain의 서브도메인으로 붙으므로 서브도메인에 DNS 등록이 필요)  
Base Domain : apiServer를 통해 통신 가능한 도메인 등록
```

Host discovery 섹션   
Add hosts에서 Provisioning type - Full image file 등록  
SSH public key : 각 노드들로 접속가능한 id_rsa 등록  
Generate Discovery ISO를 통해 discovery_image_ocp-cluster.iso 다운로드  

```
wget -O discovery_image_ocp-cluster.iso 'https://api.openshift.com/api/assisted-images/bytoken/...'
```

기본적으로 SSH key 기반 접속을 하지만 접속이 불가하다면  
`change-iso-password.sh`를 통해 해당 노드 접속 pwd 설정 가능  

```
chmod +x ./change-iso-password.sh
./change-iso-password.sh  /var/lib/libvirt/images/discovery_image_ocp-cluster.iso
```



discovery.iso를 virsh install 하기 전에 미리 cp  
```
sudo cp discovery_image_ocp-cluster.iso /var/lib/libvirt/images/discovery.iso
```

virt-install
--name : 클러스터 이름  
--ram : VRAM  
--vcpus : VPU  
--disk path : cp 했던 iso 이미지 마운트  
--graphics : vnc 포트 할당하여 진행상황 확인 가능 (port 5900~5902)  
--console : serial 타입으로 console 확인  
--noautoconosle : 생성시 콘솔 자동진입 해제  
```
virt-install \
--name ocp1 \
--ram 16384 --vcpus 4 --cpu host-passthrough \
--disk path=/var/lib/libvirt/images/ocp1.qcow2,size=120,bus=virtio \
--network network=default,model=virtio \
--cdrom /var/lib/libvirt/images/discovery.iso \
--osinfo rhel9.0 \
--graphics vnc,listen=0.0.0.0,port=5900 \
--console pty,target_type=serial \
--noautoconsole

 virt-install \
--name ocp2 \
--ram 16384 --vcpus 4 --cpu host-passthrough \
--disk path=/var/lib/libvirt/images/ocp2.qcow2,size=120,bus=virtio \
--network network=default,model=virtio \
--cdrom /var/lib/libvirt/images/discovery.iso \
--osinfo rhel9.0 \
--graphics vnc,listen=0.0.0.0,port=5901 \
--console pty,target_type=serial \
--noautoconsole

 virt-install \
--name ocp3 \
--ram 16384 --vcpus 4 --cpu host-passthrough \
--disk path=/var/lib/libvirt/images/ocp3.qcow2,size=120,bus=virtio \
--network network=default,model=virtio \
--cdrom /var/lib/libvirt/images/discovery.iso \
--osinfo rhel9.0 \
--graphics vnc,listen=0.0.0.0,port=5902 \
--console pty,target_type=serial \
--noautoconsole
```

vnc 터널 생성
```
ssh -L 5900:localhost:5900 \
    -L 5901:localhost:5901 \
    -L 5902:localhost:5902 \
    -N -f <HOST>@<IPS>
```


virt 제거
```
# 종료
sudo virsh destroy ocp1
sudo virsh destroy ocp2
sudo virsh destroy ocp3

# 제거
sudo virsh undefine ocp1 --remove-all-storage
sudo virsh undefine ocp2 --remove-all-storage
sudo virsh undefine ocp3 --remove-all-storage
```


KVM IP 확인  
```
virsh domifaddr ocp1
virsh domifaddr ocp2 
virsh domifaddr ocp3



(예시)
 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet1      :::::                ipv4         192.168.122.152/24

 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet2      :::::                ipv4         192.168.122.24/24

 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet0      :::::                ipv4         192.168.122.19/24
```

SSH 접속
```
ssh -i ~/.ssh/id_rsa core@192.168.122.19
ssh -i ~/.ssh/id_rsa core@192.168.122.24
ssh -i ~/.ssh/id_rsa core@192.168.122.152
```

진행상황 확인

```
sudo journalctl -u agent.service


# 1. 현재 네트워크 상태
nmcli device status
ip addr show
ip route show

# 2. 게이트웨이 확인 (호스트 192.168.122.1)
sudo nmcli con mod "Wired connection 1" ipv4.gateway 192.168.122.1
sudo nmcli con up "Wired connection 1"

# 3. DNS 수동 설정
sudo nmcli con mod "Wired connection 1" ipv4.dns "8.8.8.8,1.1.1.1"
sudo nmcli con up "Wired connection 1"

# 4. 테스트
ping 8.8.8.8
curl -I https://registry.redhat.io
```


virsh domain info 
```
virsh dominfo ocp1
virsh dominfo ocp2
virsh dominfo ocp3
```

```
sudo virsh start ocp1
sudo virsh start ocp2
sudo virsh start ocp3
```

현재 적용된 net-xml
```
sudo virsh net-dumpxml default
```


hosts 발견하면  

nginx 설정에 프록시 설정해둔  
VM들이랑 겹치지 않는 IP 설정해서  
192.168.122.10 -> API IP  
192.168.122.12 -> Ingress IP 설정해둠  
