# otus-hw-ci-cd

## digital-ocean
login through ssh
``` 
ssh root@138.68.85.72
```

add user githubuser
```
adduser githubuser
usermod -aG sudo githubuser
```

переключимся на нового пользователя
``` 
su githubuser
```

отныне можно другим юзером коннектиться
``` 
ssh githubuser@138.68.85.72
```

```shell
# настроим базовй брэндмаузер
ufw app list
ufw allow OpenSSH
ufw enable -y
ufw status
#>> Firewall is active and enabled on system startup


```