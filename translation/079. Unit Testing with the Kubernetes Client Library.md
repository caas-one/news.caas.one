使用Kubernetes客户端库进行单元测试
=============

如何对被 Kubernetes API 调用的代码进行单元测试？

使用 Kubernetes 客户端库可以通过模拟集群来测试代码。

作为在构建 kubernetes / minikube 时，第一批使用kubernetes / client-go 库的用户之一，我曾经详尽的设计了服务、调度单元和部署来对我的代码进行单元测试。 现在，我发现有一种更简单的方法可以用更少的代码行来做同样的事情。

我将演示如何测试一个**列出了集群中运行的所有容器映像**的简单的函数。我建议你用 GKE 或 Docker 作为桌面，并且你还需要一个 Kubernetes 集群。

### 安装程序

如果你想要运行命令并以交互方式继续操作，请克隆示例仓库 https://github.com/r2d4/k8s-unit-test-example 。

### main.go

```go
package main

import (
	"github.com/pkg/errors"
	meta_v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes/typed/core/v1"
)

// ListImages returns a list of container images running in the provided namespace
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



### 编写测试

我们可以从测试用例的定义开始，运行一些测试的框架代码。

```go
func TestListImages(t *testing.T) {
	var tests = []struct {
		description string
		namespace   string
		expected    []string
		objs        []runtime.Object
	}{
		{"no pods", "", nil, nil},
	}

	// Actual testing code goes here...
}

```

#### 发生了什么

这种编写测试的风格称为**“表驱动测试”**，是Go语言中首选的样式，实际测试代码会迭代表条目并执行必要的测试。测试代码只需要写一次，却可以适用于每种情况。我门可以注意到一些有趣的事情：

- 测试用例的定义是用匿名结构保存的，因此我们可以简洁的定义测试用例。
- 对象切片运行时，`objs`将保存我们的模拟 API 服务器保存的所有正在运行的对象，此处我们选择用一些调度单元来填充它，但其实可以在这里使用任何 Kubernetes 对象。
- 如果测试用例非常琐碎，服务器上将没有调度单元，并且不返回任何图像。

### 测试回路

让我们来为每个测试用例都写一段实际测试代码。

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

在这里我们也可以注意到一些有趣的事情：

- 文件`t.Run` 的功能是执行子测试，那么为什么要使用子测试？

- - 你可以使用`-run` 命令运行特定的测试用例进行测试。

  - 你可以设置和清理测试代码
  - 子测试是同时运行测试用例的入口（这篇文章将不涉及）

- 实际结果和预期结果都与 `cmp.diff` 不同。Diff 返回的两个值之间差异是可读的，当且仅当输入值和选项相等且为真时，Diff返回空字符串。

`fake.NewSimpleClientset` 返回一个使用提供的对象进行响应的客户端集。它由一个非常简单的对象跟踪器支持，该跟踪器按原样处理创建、更新和删除操作，而不应用任何验证和默认值。

### 测试用例

我们来创建一个调度单元助手函数，它将帮助我们提供一些测试的调度单元。由于我们关心的是命名空间和镜像，所以我们可以创建一个可以基于这些参数创建新的调度单元的助手。

```go
func pod(namespace, image string) *v1.Pod {
	return &v1.Pod{ObjectMeta: meta_v1.ObjectMeta{Namespace: namespace}, Spec: v1.PodSpec{Containers: []v1.Container{{Image: image}}}}
}
```

我们来写三个单元测试。第一个方法能确保我门获取所有镜像，这时我们需要使用特殊的命名空间值 `“”` 来列出命名空间中所有的调度单元。

```go
{"all namespaces", "", []string{"a", "b"}, []runtime.Object{pod("correct-namespace", "a"), pod("wrong-namespace", "b")}}
```

第二种方法能确保我们按名称空间正确筛选，忽略调度单元中的 `wrong-namespace` 。

```go
{"filter namespace", "correct-namespace", []string{"a"}, []runtime.Object{pod("correct-namespace", "a"), pod("wrong-namespace", "b")}}
```

第三种方法能确保如果所需的命名空间中没有调度单元，则不会返回任何内容。

```go
{"wrong namespace", "correct-namespace", nil, []runtime.Object{pod("wrong-namespace", "b")}}
```

把上面这些代码放在一起。

```go
func TestListImages(t *testing.T) {
	var tests = []struct {
		description string
		namespace   string
		expected    []string
		objs        []runtime.Object
	}{
		{"no pods", "", nil, nil},
		{"all namespaces", "", []string{"a", "b"}, []runtime.Object{pod("correct-namespace", "a"), pod("wrong-namespace", "b")}},
		{"filter namespace", "correct-namespace", []string{"a"}, []runtime.Object{pod("correct-namespace", "a"), pod("wrong-namespace", "b")}},
		{"wrong namespace", "correct-namespace", nil, []runtime.Object{pod("wrong-namespace", "b")}},
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

### 原文链接
[原文作者：@mattrickard]
[原文链接：https://matt-rickard.com/kubernetes-unit-testing/](https://matt-rickard.com/kubernetes-unit-testing/)
