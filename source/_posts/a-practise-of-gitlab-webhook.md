title: 记一次gitlab webhook使用实践
date: 2016-09-04 13:47:44
tags:
- webhook
- git
---

## 背景
最近我们团队将git服务器全部迁移到gitlab上，出于某些考虑，我们将所有项目的master分支设为受保护状态，小组成员只能对dev或者其他分支推送代码。当需要发布版本时，都需要组长将代码合并到master，这样当频率一高，组长必然会不断得被打断工作，从而影响效率。

刚开始时我想利用git alias或者批处理来解决这件时，然而细想，如果还是需要我来敲一行命令或者打开批处理，仍然没有解决问题，我需要的是gitlab的消息通知，从而让我的本地服务来完成这件事。恰巧在在某个广告邮件中，让我知道webhook这个东西。

## 什么是webhook
那么，webhook是个什么东西？从字面意思上看，叫web钩子。实际上可以把它理解为回调，或者委托，或者事件通知，归根揭底它就是一个消息通知机制。当gitlab触发某个事件时，它会通知你的所配置的http服务，然后你想干嘛干嘛吧。gitlab所支持的webhook事件类型，包括以下这些
![webhook-triggers](/img/webhook-triggers.png)

## 如何使用
首先我们需要添加一个webhook，具体怎么添加不多说了，在项目设置里面大家界面多找找就知道了，我的设置如下
![webhook-setting](/img/webhook-setting.png)

接着，我们只要实现我们的9999端口的api服务就行了，我的demo在这里，用的golang
```golang
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/exec"
	"strings"
)

// HelloServer hello world, the web server
func HelloServer(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "Hi there, I love %s!", req.URL.Path[1:])

	decoder := json.NewDecoder(req.Body)
	var context PushContext
	err := decoder.Decode(&context)
	if err != nil {
		panic(err)
	}

	folder := context.Project.Name + "-webhook"

	// clone git repo
	var existRepo bool
	existRepo, err = exists(folder)
	if err != nil {
		panic(err)
	}
	if !existRepo {
		exeCmd("cmd /C git clone " + context.Project.GitHTTPURL + " " + folder)
	}

	// cd repo
	// checkout dev
	// pull dev
	// checkout master
	// pull master
	// merge dev
	// push master
	exeCmd("cmd /C cd " + folder + "&&git checkout dev&&git pull&&git checkout master&&git pull&&git merge dev&&git push")
}

func main() {
	http.HandleFunc("/test", HelloServer)
	log.Fatal(http.ListenAndServe(":9999", nil))
}

func exists(path string) (bool, error) {
	_, err := os.Stat(path)
	if err == nil {
		return true, nil
	}
	if os.IsNotExist(err) {
		return false, nil
	}
	return true, err
}

func exeCmd(cmd string) {
	fmt.Println("command is ", cmd)
	// splitting head => g++ parts => rest of the command
	parts := strings.Fields(cmd)
	head := parts[0]
	parts = parts[1:len(parts)]

	out, err := exec.Command(head, parts...).Output()
	if err != nil {
		fmt.Printf("%s", err)
	}
	fmt.Printf("%s", out)
}

// PushContext post body
type PushContext struct {
	ObjectKind string  `json:"object_kind"`
	Ref        string  `json:"ref"`
	UserID     int     `json:"user_id"`
	UserName   string  `json:"user_name"`
	Project    Project `json:"project"`
}

// Project project
type Project struct {
	Name       string `json:"name"`
	GitHTTPURL string `json:"git_http_url"`
}

```

## 使用设想
知道了webhook，我们就可以利用它来做一些有意义的事情，比如持续集成。我们已经有了一个.NET发布平台，然是仍然需要手动去点两下按钮，后面可以通过webhook来实现一旦master push就自动发布的功能。

另外，对于我部署在github的静态blog，也可以实现自动发布，提高我的工作效率。
