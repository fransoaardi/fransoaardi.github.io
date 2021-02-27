---
layout: post
title: AWS EBS volume mount
date: 2021-02-09 17:30:00 +0900
categories: [Wiki]
tags: [aws]
toc: true
comments: true
---

# introduction

- aws ec2 사용 중, filesystem 이 꽉찬 서버가 있어서, mount 되어있었던 volume (Elastic Block Storage, 이하 EBS) 의 사이즈를 키울 일이 있었다.

- 두가지 특이한 점이 있었다. 첫번째는 이 서버에 남은 공간이 없어서 `growpart` 가 불가능했던점, 두번째는 linux 의 filesystem type 을 체크를 안해서 생겼던 문제이다.

# progress

## 1. filesystem grow, expand 작업 

- 일단 문제의 ec2 에 매핑된 storage 의 할당을 늘렸다.

- 특이하게 상태가 `in-use` 에서 `in-use optimizing(0%)` 로 변경이되면서 100% 까지 천천히 올라갔다.

- disk size 를 확인한다.

```bash
ubuntu@:~$ df -hT
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/xvda1     ext4      7.7G  7.7G     0 100% /
```

- 서버에서 block 상태를 확인한다

```bash
$ lsblk
ubuntu@:~$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0    0   20G  0 disk
└─xvda1 202:1    0    8G  0 part /

$ lsblk -f
NAME    FSTYPE   LABEL           UUID                                 MOUNTPOINT
xvda
└─xvda1 ext4     cloudimg-rootfs abcdefgh-425e-44bb-9de8-7d7877d31328 /
```

- `8G -> 20G` 로 올렸는데 아직 8G 로 남아있다.
- `ext4` 혹은 `xfs` 를 본 적이 있다. (현재는 `ext4`)

### filesystem 이 xfs 일때

- grow xvda1

```bash
$ sudo growpart /dev/xvda 1
```

- expand xfs filesystem

```bash
$ sudo yum install xfsprogs
$ sudo xfs_growfs -d /
# because xvda1 mount on /
```

- `when filesystem is already created`: 파일시스템이 구성된 상태

```bash
$ sudo file -s /dev/xvdf 
dev/xvdf: SGI XFS filesystem data (blksz 4096, inosz 512, v2 dirs)
```

- mount

```bash
$ sudo mount /dev/xvdf ~/path/to/mount
```

### filesystem 이 ext4 일때

- **주의: ext4 filesystem 에 xfs_growfs 쓰면 아래와 같이 에러가 발생한다.**

```bash
ubuntu@:~$ sudo xfs_growfs -d /
xfs_growfs: specified file ["/"] is not on an XFS filesystem
```

- grow

```bash
$ sudo growpart /dev/xvda 1
```

- expand

```bash
$ sudo resize2fs /dev/xvda1
resize2fs 1.44.1 (24-Mar-2018)
Filesystem at /dev/xvda1 is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 3
The filesystem on /dev/xvda1 is now 5242619 (4k) blocks long.
```

## 2. 남은공간이 없는 경우

> 출처: https://aws.amazon.com/ko/premiumsupport/knowledge-center/ebs-volume-size-increase/

- EBS 볼륨에서 루트 파티션이나 루트 파일 시스템을 확장할 때 디바이스에 남은 공간 없음 오류를 방지하려면 메모리에 상주하는 임시 파일 시스템인 tmpfs를 사용합니다. tmpfs 파일 시스템을 /tmp 탑재 지점 아래에 탑재한 다음, 루트 파티션 또는 루트 파일 시스템을 확장합니다.

- 예를들어 아래와 같은 에러 메세지를 만난 경우이다, df -h 해보면 Used 100% 이다.

```bash
$ sudo growpart /dev/nvme0n1 1
/bin/growpart: line 248: /tmp/growpart.fklt5u/dump.out: No space left on device
FAILED: failed to dump sfdisk info for /dev/nvme0n1
```

- 이 경우, tmpfs 파일 시스템을 /tmp 에 잠깐 탑재하고, 확장하고, 다시 unmount 하는 방식을 취한다.

```bash
$ sudo mount -o size=10M,rw,nodev,nosuid -t tmpfs tmpfs /tmp
```

- 모든 작업이 끝나고 늘렸던 /tmp 를 unmount 한다.

```
$ sudo umount /tmp
```

## 참고

### how to make mounted EBS automatically mount on instance reboot

- ebs mount 한 내용이 instance reboot 했을때 적용이 되지 않을 수 있어, 부팅시 적용되도록 설정을 하는 방법이다. 

- `ext4` 에서 해보진 않았고, 이전에 진행했던 `xfs` 기준으로 작성한다. (`ext4` 도 방법이 아래의 reference 에 제공된다)

- check UUID with blkid

```bash
$ blkid
/dev/xvda1: LABEL="/" UUID="here-comes-uuid-1" TYPE="xfs" PARTLABEL="Linux" PARTUUID="here-comes-part-uuid"
/dev/xvdf: UUID="here-comes-uuid-2" TYPE="xfs"
```

- make mount automatically on reboot : `/etc/fstab`

```bash
UUID=here-comes-uuid-1     /                             xfs    defaults,noatime  1   1
UUID=here-comes-uuid-2     /home/ec2-user/path/to/mount  xfs    defaults,nofail   0   2
```

- testing umount/mount

```bash
$ sudo umount ~/path/to/mount
$ sudo mount -a
```

- mount check

```bash
$ mount
$ df
```

# result

- lsblk 결과: 20G 로 늘어난것을 확인할 수 있다.

```bash
ubuntu@:~$ lsblk
xvda    202:0    0   20G  0 disk
└─xvda1 202:1    0   20G  0 part /
```

# references

- 전반적인 guide
    - https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html

- disk storage 가 꽉찼을때 ebs volume increase 하는 과정
    - https://aws.amazon.com/ko/premiumsupport/knowledge-center/ebs-volume-size-increase/
