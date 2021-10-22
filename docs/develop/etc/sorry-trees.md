---
layout: default
title: 미안해 컴퓨터야
parent: etc
grand_parent: 개발
created_at: 2018.08.09
print_title: true
share_enable: true
permalink: develop/etc/sorry-computers
---

그러니까, 이론적으로 이 코드는 'whojes' 를 찍을 수 있는데, 

```go
func main(){
	var a int
	b := &a
	go func(){
		for {
			*b = 1
		}
	}()
	go func(){
		for{
			*b = 2
		}
	}()
	for {
		if a == 1 && a == 2 {
			fmt.Println("whojes")
		}
	}
}

```
그러니까 a 는 1이면서 동시에 2일 수 있다는 건데, 현대 cpu 나 컴파일러중에 이런게 되게 하는게 있으려나? 
