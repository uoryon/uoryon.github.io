---
layout: post
title:  "学习实现 k8s TPR"
date:   2017-06-13 11:23:02 +0800
categories: k8s
comments: true
---

总结一下写 k8s TPR 遇到的坑，以及如何写一个 TPR。

TPR = ThirdPartyResource 虽然马上就会被 deprecated 了，但是整个过程还是能有一些借鉴意义的。

使用 TPR，可以在 k8s apiserver 里面注册一个资源，用户可以使用 yaml 描述这种资源。TPR 维护者可以通过 apiserver 感知到资源的创建、更新和删除。
<!--more-->
- - - -

### MongoOperator
github: [GitHub - uoryon/mongo-operator](https://github.com/uoryon/mongo-operator)

用了一些时间完成了上面这个作品，一个 Mongo 的 Operator，达到的效果就是用户通过一个 yaml，能够直接启动一个已经配置好的 Mongo Replica Set 集群。然后用户就可以专注业务了，mongo 的灾难恢复可以交给我来做了，做得更好的话，应该能够自带监控以及自动备份。现在的灾难恢复也比较粗糙，以后希望能有机会可以变得更好吧。

##### 与 apiserver 通信
与 k8s 相关的交互应该都是要从这里开始的。我使用的库是 [GitHub - kubernetes/client-go: Go client for Kubernetes.](https://github.com/kubernetes/client-go) 。
首先需要提供 kubeconfig。`mac`本地用`minikube` 启动的话，应该是 `~/.kube/config`。如果你的这项服务是跑在 k8s 集群里面，直接调用`k8s.io/client-go/rest.InClusterConfig`方法可以取得。`k8s.io/client-go/tools/clientcmd.BuildConfigFromFlags`也能得到 `Config`对象，将这个对象传入 `k8s.io/client-go/kubernetes.NewForConfig`就可以得到 `Clientset`了。
```
// Clientset contains the clients for groups. Each group has exactly one
// version included in a Clientset.
type Clientset struct {
	*discovery.DiscoveryClient
	*corev1.CoreV1Client
	*appsv1beta1.AppsV1beta1Client
	*authenticationv1.AuthenticationV1Client
	*authenticationv1beta1.AuthenticationV1beta1Client
	*authorizationv1.AuthorizationV1Client
	*authorizationv1beta1.AuthorizationV1beta1Client
	*autoscalingv1.AutoscalingV1Client
	*autoscalingv2alpha1.AutoscalingV2alpha1Client
	*batchv1.BatchV1Client
	*batchv2alpha1.BatchV2alpha1Client
	*certificatesv1beta1.CertificatesV1beta1Client
	*extensionsv1beta1.ExtensionsV1beta1Client
	*policyv1beta1.PolicyV1beta1Client
	*rbacv1beta1.RbacV1beta1Client
	*rbacv1alpha1.RbacV1alpha1Client
	*settingsv1alpha1.SettingsV1alpha1Client
	*storagev1beta1.StorageV1beta1Client
	*storagev1.StorageV1Client
}
```

通过这个对象，就可以操作你的资源了。当然可能需要相应的权限，跑在 k8s 内部的话，通过 `ServiceAccount`的方式就行了。

接下来就是创建我的 TPR 资源了。
```
tpr := &v1beta1.ThirdPartyResource{
	ObjectMeta: metav1.ObjectMeta{
		Name: proto.MgorsKind + "." + proto.GroupName,
	},
	Versions: []v1beta1.APIVersion{
		{Name: proto.MgoOpGroupVersion.Version},
	},
	Description: "A Mongo replica set",
}
_, err := clientset.ExtensionsV1beta1().ThirdPartyResources().Create(tpr)
return err
```
k8s 一些操作是异步的，比如删除 pod，以及创建 TPR，当你发出调用说创建的时候，那一刻很可能 TPR 还没有创建好。调用 `k8s.io/apimachinery/pkg/util/wait`里面的方法等待资源准备完毕即可。里面的方法都挺好用的。

创建完这个 TPR 后，用户就可以开始创建这种资源了，因为我们的 operator 是类似 controller-manager 的性质，所以就需要 list-watch 这个资源了，我们不能直接使用 k8s 的那个 `Clientset`，k8s 的 client 非常神奇，有一个库是 [GitHub - kubernetes/apimachinery](https://github.com/kubernetes/apimachinery)，这是 api 用的。这个库比较大，要讲的话就太多了，简单来说就是一个资源和一个 gvk（group, version, kind）一一对应，要把资源和 gv 组册到 schema 中，才能够序列化成功。因为 `Clientset`不会注册我们管理 TPR 的那个对象。
下面是我们对象的声明形式：
```
// Mgors is the struct for MgoRSKind
type Mgors struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata"`

	Spec   MgorsSpec `json:"spec"`
	Status MgoStatus `json:"status"`
}

// MgorsList Mgors's List
type MgorsList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`
	Items           []Mgors `json:"items"`
}
```
记住 Kind 的名字要跟 Struct 对应上。例如上面是 `Mgors`，kind 需要是 `mgors`。然后我们可以开始构造我们想要的客户端了。
看这个地方。[mongo-operator/client.go at master · uoryon/mongo-operator · GitHub](https://github.com/uoryon/mongo-operator/blob/master/pkg/client/client.go#L70)这样就能够着我们想要的客户端了。代码挺多，就不贴了，去 Github 看吧，比较关键的是那个 `AddToScheme`。关键的一步是
```
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(MgoOpGroupVersion,
		&Mgors{},
		&MgorsList{},
	)
	metav1.AddToGroupVersion(scheme, MgoOpGroupVersion)
	return nil
}
```
这样我们就能序列化这些对象，这个 `client` 就可用了，我们能够使用 `list-watch`机制来管理这些资源了。
看这个调用即可，[mongo-operator/mgors_controller.go at master · uoryon/mongo-operator · GitHub](https://github.com/uoryon/mongo-operator/blob/master/pkg/controller/mgors_controller.go#L43)，算是比较简单。需要注意的是，`resources`要用 kind 的复数形式，我们的是 `mgors`，就需要转成`mgorses`。

了解了上面这些，基本可以开始研发自己的 `CustomResource`了。先介绍到这里。
