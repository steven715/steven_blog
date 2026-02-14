# VMware Network Type

## Bridged

VMs 能獨立於 host ，擁有自己的IP，並與外部連線。

## NAT

VMs 會透過 VMWare 的虛擬網卡 vEthernet 來與外部連線。

## Host-only

VMs 都在同一個網段，VM 之間網路互通，但沒有對外能力。

# Docker Network Type

## bridge

類似 NAT，host 與 container 的連線透過 docker 的虛擬網卡 docker0 來與外部連線。

## host

Container 直接與 host 共用網卡。

## none

Container 只有 loopback，沒有連網功能。
