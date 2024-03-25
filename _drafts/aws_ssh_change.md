

# Introduction
- ec2 의 keypair 를 변경하려고 한다.
- ec2 instance 생성시에 예를들어 first_key.pem 으로 적용했었는데, 새로운 pem(`second_key.pem`) 으로 변경하고싶다. 

# References
- https://aws.amazon.com/ko/premiumsupport/knowledge-center/user-data-replace-key-pair-ec2/

# Howto
- 새로운 pem 생성
- running 중이던 ec2 instance stop
- 아래의 script 를 `사용자 데이터 보기/변경` 에 입력후 재시작, `${username}`, `${KeyPairs}` 변경 필요.
- `KeyPairs` 는 아래와 같이 cmd 통해서 확인해볼 수 있음
> `$ ssh-keygen -y -f /Users/hmc/env/aws_pem/second_key.pem`
```
Content-Type: multipart/mixed; boundary="//"
MIME-Version: 1.0

--//
Content-Type: text/cloud-config; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
Content-Disposition: attachment; filename="cloud-config.txt"

#cloud-config
cloud_final_modules:
- [users-groups, once]
users:
  - name: {username}
    ssh-authorized-keys: 
    - ${KeyPairs}
```

- 재시작 해도, 아래와 같이 인스턴스 소유자의 키페어 부분은 그대로 있음 
![image](https://github.hmckmc.co.kr/storage/user/57/files/d9db6480-daf8-11ea-9aa1-a5acd29b5b55)

- 다만, 기존 keypair(`first_key.pem`) 과 변경한 keypair(`second_key.pem`) 둘 다 접근 가능
- 위에 script 입력한 것은 보안적으로 취약할 수 있으니, 동일한 instance 를 stop, 내용을 지운 뒤 다시 다시 start 함
- instance 에 접근해서 `~/.ssh/authorized_keys` 에서 기존 pem (`first_key.pem`) 을 제거한 뒤 저장

# Conclusion
- 기존 pem(`first_key.pem`) 으로 접근은 막고, 새로운 pem(`second_key.pem`) 으로의 접근은 가능하게 변경하는데 성공했다. (목적 달성)
- pem 파일 변경해도 ec2 instance 상세정보에 keypair 부분에 이름이 바뀌진 않는다. (meta 정보에 기록된 내용이 `~/.ssh/authorized_keys` 와 동기화 될거라고 생각한게 잘못이다..)
- `aws system managers` 라는게 있는데, ec2 instance 를 중앙관리 하는것인것 같은데, 이에 등록해서 변경하면 될지는 테스트 해보지 않았다. 
