# This is just a quick set of notes that I made while exploring the new Bottlerocket OS:

I started with a simple eksctl config:

Sample eksworkshop.yaml:
```
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop-eksctl
  region: us-west-2
  version: '1.15'

nodeGroups:
- name: bottlerocket-ng1
  desiredCapacity: 3
  ami: auto-ssm
  amiFamily: Bottlerocket
  iam:
     attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
  bottlerocket:
    enableAdminContainer: true
    settings:
      motd: "Hello from eksctl!"
  ssh:
    allow: true
    publicKeyName: eksworkshop

secretsEncryption:
  keyARN: arn:aws:kms:us-west-2:651426287273:key/09b590a9-8502-47ad-8098-f9610858483c
```


#### Create EKS cluster using Bottlerocket nodes:
```
eksctl create cluster -f eksworkshop.yaml
```
#### after cluster creation:
```
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.6/config/v1.6/aws-k8s-cni.yaml
kubectl edit -n kube-system daemonset kube-proxy
```
#### add the bottom two options:
```
      containers:
      - command:
        - kube-proxy
        - --v=2
        - --config=/var/lib/kube-proxy-config/config
        - --conntrack-max-per-core=0
        - --conntrack-min=0
```        
#### Run a container to test
```
kubectl run -i -t busybox --image=busybox --restart=Never
```

#### visit session manager:
https://us-west-2.console.aws.amazon.com/systems-manager/session-manager/sessions?region=us-west-2

#### start session with one of the eks worker nodes:
```
apiclient --help
apiclient -u /settings # (to get settings)
```
#### patch settings changes to stage changes:
```
apiclient -m PATCH -u /settings -d '{"motd": "Hello from Brent!"}'
```
#### view with a get to /tx:
apiclient -u /tx

- make a POST call to /tx/commit to make the changes live:
- make a POST call to /tx/apply to trigger an external settings applier that will apply the changes and restart any necessary services
- make a POST call to /tx/commit_and_apply to do both:
```
apiclient -m POST -u /tx/commit_and_apply
```

#### ssh in to the admin container:
```
until ssh ec2-user@54.185.175.90; do sleep 1; done
```

#### after ssh'ing in, sudo to become root
```
sudo sheltie
```

#### see release version:
```
cat /etc/os-release
```

# Updates:
start with downgrading to older release and rebooting:
```
updog update  -i 0.3.0 --reboot
```
Show active/next a/b volumes:
```
signpost status
```
Output:
```
bash-5.0# signpost status
OS disk: /dev/nvme0n1
Set A:   boot=/dev/nvme0n1p2 root=/dev/nvme0n1p3 hash=/dev/nvme0n1p4 priority=1 tries_left=0 successful=true
Set B:   boot=/dev/nvme0n1p6 root=/dev/nvme0n1p7 hash=/dev/nvme0n1p8 priority=2 tries_left=0 successful=true
Active:  Set B
Next:    Set B
```
Show that update is available:
```
updog check-update
```
Output:
```
bash-5.0# updog check-update
aws-k8s-1.15 0.3.1
```
Perform update:
```
updog update
```
Output:
```
bash-5.0# updog update
Starting update to 0.3.1
Update applied: aws-k8s-1.15 0.3.1
```
Show new active/next a/b volumes:
```
signpost status
```
Output:
```
bash-5.0# signpost status
OS disk: /dev/nvme0n1
Set A:   boot=/dev/nvme0n1p2 root=/dev/nvme0n1p3 hash=/dev/nvme0n1p4 priority=2 tries_left=1 successful=false
Set B:   boot=/dev/nvme0n1p6 root=/dev/nvme0n1p7 hash=/dev/nvme0n1p8 priority=1 tries_left=0 successful=true
Active:  Set B
Next:    Set A
```
Update with reboot (or just reboot):
```
updog update --reboot
signpost status
```
Output:
```
bash-5.0# signpost status
OS disk: /dev/nvme0n1
Set A:   boot=/dev/nvme0n1p2 root=/dev/nvme0n1p3 hash=/dev/nvme0n1p4 priority=2 tries_left=0 successful=true
Set B:   boot=/dev/nvme0n1p6 root=/dev/nvme0n1p7 hash=/dev/nvme0n1p8 priority=1 tries_left=0 successful=true
Active:  Set A
Next:    Set A
```

# Other useful info:
- journalctl -a 
- containerd config dump
- systemctl status kubelet

# security:
 - immutable rootfs
 - uses dm-verity for integrity checking of the underlying block device
 - mounted read-only and can't be remounted by a userspace process
```
mount | head -n1
```
- kernel is configured to restart if corruption is detected (for example modifying underlying block device)
- /etc is stateless tmpfs, so modifying things like /etc/resolv.conf is pointless
- Secondary writable filesystem is mounted to /local with bindmounts for /opt and /var. 
- No shell or interpreters installed:
```
echo $SHELL
ls -la /bin/bash
```
- selinux enabled in enforcing mode
Once the policy is loaded, there is no way to unload it.
(superpowered = true means those containers are run permisively with the super_t domain)
- Actions not allowed by the policy will only be logged, not blocked.
(currently policy is incomplete, so all domains are permissive, not strict)
