![img](assets/Centos Stream 9/1671354551139-23e27274-1f6b-4ce2-8fc5-78e9f9d2d2bb.png)

![img](assets/Centos Stream 9/1671354598598-af243df2-77fc-4ee4-b252-1f16ac6f4236.png)

![img](assets/Centos Stream 9/1671354623362-fa26d143-d3c5-459a-bb4c-836a10e78270.png)

![img](assets/Centos Stream 9/1671354828663-e461b537-7054-4a4f-aadb-02098629a165.png)

![img](assets/Centos Stream 9/1671354887285-7d6591d4-6352-4f71-a83e-82e911d9e4d0.png)

系统默认不允许root用户远程登录，需要修改

```shell
# vim /etc/ssh/sshd_config
```

![img](assets/Centos Stream 9/1671355196009-111c2c89-2905-4527-b57f-f56a132975c4.png)



ok，然后关机，先拍快照

```shell
# cd /etc/NetworkManager/system-connections/
```

![img](assets/Centos Stream 9/1671356104603-ca227382-b986-406a-9757-472e83f2f007.png)

```shell
# nmcli con up ens33
```