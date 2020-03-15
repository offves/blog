> 后台启动
```shell
nohup java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 -jar neteasecloudmusic-0.0.1-SNAPSHOT.jar --server.prot=8080 >service.log 2>&1 &
```


> ssh proxy:
```shell
ProxyCommand nc -X 5 -x 127.0.0.1:7891 %h %p
```

