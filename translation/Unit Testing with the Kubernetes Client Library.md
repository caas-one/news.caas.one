# 如何进行单元测试代码来调用Kubernetes API

使用kubernetes客户机库可以帮助您模拟出一个集群来测试您的代码。

在构建[kubernetes/minikube](https://github.com/kubernetes/minikube)时，作为[kubernetes/client-go ](https://github.com/kubernetes/client-go)库的第一个消费者之一，services、pods和deployments构建了详细的模拟，以单元测为例，现在，有一种更简单的方法可以用更少的代码行来做同样的事情。 

我将演示如何测试一个简单的函数，该函数列出了集群中运行的所有容器镜像。你需要一个Kubernetes集群，我建议用GKE或桌面版的Docker。

## Setup
克隆示例存储库[https://github.com/r2d4/k8s-unit-test-example ](https://github.com/r2d4/k8s-unit-test-example )，如果要运行命令并以交互方式继续操作。

###main.go

```go
import  ( 
    		"github.com/pkg/errors" 
			meta_v1 "k8s.io/apimachinery/pkg/apis/meta/v1" 
			"k8s.io/client-go/kubernetes/typed/core/v1" 
) 
//ListImages返回在提供的命名空间中运行的容器映像的列表
func ListImages(client v1.CoreV1Interface, namespace string) ([]string, error) { 
		pl, err := client.Pods(namespace).List(meta_v1.ListOptions{})
 		if err != nil { 
			return nil, errors.Wrap(err, "getting pods") 
		} 
		var images []string 
		for _, p := range pl.Items { 
			for _, c := range p.Spec.Containers {
 				images = append(images, c.Image) 
			} 
		}
 		return images, nil
 }
```
 
 
## Writing the Tests
从测试用例的定义开始，以及一些运行测试的框架代码。

```go
func TestListImages(t *testing.T) {
 	var tests = []struct { 
		description string 
		namespace string 
		expected []string 
		objs []runtime.Object 
	}
	{ 
		{"no pods", "", nil, nil}, 
	} 
	// 在这写需要测试的代码
}
```

## What's Happening


这种编写测试的风格称为“表驱动测试”，在Go中，这是首选的样式。实际的测试代码迭代表条目并执行必要的测试。测试代码只写一次，用于每种情况。需要注意一些事情： 

* 用于保存测试用例定义的匿名结构。它们允许我们简洁地定义测试用例。


* 运行时对象切片objs将保存所有希望我们的模拟API服务器保存的运行时对象。将用一些pods填充,但是您可以在这里使用任何kubernetes对象。
    
* 琐碎的测试用例，服务器上没有pods不应返回任何image。


## Test Loop
填写将为每个测试用例运行的实际测试代码。

```go
for _, test := range tests { 
	t.Run(test.description, func(t *testing.T) { 
		client := fake.NewSimpleClientset(test.objs...) 
		actual, err := ListImages(client.CoreV1(), test.namespace) 
		if err != nil {
 			t.Errorf("Unexpected error: %s", err)
 			return 
		} 
		if diff := cmp.Diff(actual, test.expected); diff != "" { 
			t.Errorf("%T differ (-got, +want): %s", test.expected, diff) 			
			return 
		} 
	}) 
}
```
需要注意的一些有趣的事情，
t.run运行执行子测试。为什么使用子测试？

* 可以使用-run flg to go test

* 可以做设置和拆卸子测试是并行运行测试用例的入口（这里不做）

* 实际结果和预期结果与cmp.diff不同。diff返回两个值之间差异的人类可读报告。如果相同的输入值和选项的equal返回true，则返回空字符串。

fake.newsImpleClientSet返回将使用提供的对象响应的客户端集。 它由一个非常简单的对象跟踪器提供支持，该跟踪器按原样处理增加、更新和删除操作，不应用任何验证和/或默认值。


## Test Cases

  创建一个pod函数，它将帮助提供一些pod供我们测试。既然我们关心namespace和image,它基于这些参数创建新的pod.
  
```
func pod(namespace, image string) *v1.Pod { 
		return  &v1.Pod{ObjectMeta: meta_v1.ObjectMeta{Namespace: namespace}, Spec: v1.PodSpec{Containers: []v1.Container{{Image: image}}}} 
}
```

  写三个单元测试。如果使用特殊的namespace值列出所有namespace中的pods，第一个方法将确保获取所有image。
  
```
{"all namespaces", "", []string{"a", "b"}, []runtime.Object{pod("correct-namespace", "a"), pod("wrong-namespace", "b")}}
```

  第二种情况将确保按namespace正确筛选，忽略错误namespace中的pod。
```
{"filter namespace", "correct-namespace", []string{"a"}, []runtime.Object{pod("correct-namespace", "a"), pod("wrong-namespace", "b")}}
```

  第三种情况将确保如果所需的namespace中没有pods，则不会返回任何内容。
```
{"wrong namespace", "correct-namespace", nil, []runtime.Object{pod("wrong-namespace", "b")}}
```
#### Putting it all together
```go
func TestListImages(t *testing.T) { 
	var tests = []struct { 
		description string 
		namespace string 
		expected []string
 		objs []runtime.Object 
	}
	{ 
		{"no pods", "", nil, nil}, 
		{"all namespaces", "", []string{"a", "b"},[]runtime.Object{pod("correct-namespace", "a"), pod("wrong-namespace", "b")}}, 
		{"filter namespace", "correct-namespace", []string{"a"},[]runtime.Object{pod("correct-namespace", "a"), pod("wrong-namespace", "b")}}, 
		{"wrong namespace", "correct-namespace", nil,[]runtime.Object{pod("wrong-namespace", "b")}}, 
	} 
	for _, test := range tests { 
		t.Run(test.description, func(t *testing.T) { 
			client := fake.NewSimpleClientset(test.objs...) 
			actual, err := ListImages(client.CoreV1(), test.namespace) 
			if err != nil { 
				t.Errorf("Unexpected error: %s", err)
				return 
			} 
			if diff := cmp.Diff(actual, test.expected); diff != "" { 
				t.Errorf("%T differ (-got, +want): %s", test.expected, diff)
 				return 
			}
		 })
	 }
 } 
```
Matt Rickard

[https://matt-rickard.com/kubernetes-unit-testing/](https://matt-rickard.com/kubernetes-unit-testing/)
