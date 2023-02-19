# 如何删除 nexus 私服上的 Docker 镜像

当 nexus 的 docker 镜像越来越多时，老的镜像就需要删除，用来释放空间

找了一个简易的办法， 首先下载 [regclient](https://github.com/regclient/regclient) 下的 `regctl` 命令
然后创建一个类似如下的脚本来删除老的

```shell
#!/bin/sh
export PATH=/root/bin:$PATH
registry="nexus.yourdomain.com"
cutoff="$(date -d -30days '+%s')"
for repo in $(regctl repo ls "$registry" | grep  -E '(staff-center|lczq-ias|ims|crm|income-voucher|wechat-work|grp-arch)'); do
  # The "head -n -5" ignores the last 5 tags, but you may want to sort that list first.
  for tag in $(regctl tag ls "$registry/$repo" |grep 'test-' | head -n -30); do
    # This is the most likely command to fail since the created timestamp is optional, may be set to 0,
    # and the string format might vary.
    # The cut is to remove the "+0000" that breaks the "date" command.
    created="$(regctl image config "$registry/$repo:$tag" --format '{{.Created}}' | cut -f1,2,4 -d' ')"
    createdSec="$(date -d "$created" '+%s')"
    # both timestamps are converted to seconds since epoc, allowing numeric comparison
    if [ "$createdSec" -lt "$cutoff" ]; then
      # next line is prefixed with echo for debugging, delete the echo to run the tag delete command
      # get digest
      digest=$(regctl image digest ${registry}/${repo}:${tag})
      regctl image delete "$registry/$repo:$tag@${digest}"
    fi
  done
done
```

我这里做了一些过滤，直把自有的一些镜像删除，公共的镜像不删除
