---
title: "Make Rabbitmq Openshift Operator"
date: 2021-04-20T09:24:32+08:00
draft: false
---

### 制作 rabbitmq openshift operator
- 以 rabbitmq-operator github 官方项目为基础
- https://github.com/rabbitmq/cluster-operator

### 命令行工具
1.  kustomize
> [下载地址](https://github.com/kubernetes-sigs/kustomize/releases)
2. operator-sdk 
> [下载地址](https://github.com/operator-framework/operator-sdk/releases)
3. opm
> [下载地址](https://github.com/operator-framework/operator-registry/releases)

### 制作过程
- 本文档以 rabbitmq 1.6.0 为基础制作 operator，[link](https://github.com/rabbitmq/cluster-operator/tree/v1.6.0)
---
#### 下载源码
1. 从官方 releases 页面下载 1.6.0 源码
2. 解压后改名为 rabbitmq-operator

#### 修改部分源码
1. 删除 rabbitmq-operator/.github 目录，此目录是官方 github action 部分的代码，对其他项目无用（也可以不删除）
2. 在 rabbitmq-operator/internal/resource 目录新建 anyuid_role_binding.go 文件，将下面代码复制到文件中，代码是根据 role_binding.go 文件修改来的，目的是解决 openshift scc 权限问题
```
// RabbitMQ Cluster Operator
//
// Copyright 2020 VMware, Inc. All Rights Reserved.
//
// This product is licensed to you under the Mozilla Public license, Version 2.0 (the "License").  You may not use this product except in compliance with the Mozilla Public License.
//
// This product may include a number of subcomponents with separate copyright notices and license terms. Your use of these subcomponents is subject to the terms and conditions of the subcomponent's license, as noted in the LICENSE file.
//

package resource

import (
	"fmt"
	rbacv1 "k8s.io/api/rbac/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"

	rabbitmqv1beta1 "github.com/rabbitmq/cluster-operator/api/v1beta1"
	"github.com/rabbitmq/cluster-operator/internal/metadata"
)

const (
	anyuidRoleBindingName = "anyuid"
)

type AnyuidRoleBindingBuilder struct {
	Instance *rabbitmqv1beta1.RabbitmqCluster
	Scheme   *runtime.Scheme
}

func (builder *RabbitmqResourceBuilder) AnyuidRoleBinding () *AnyuidRoleBindingBuilder {
	return &AnyuidRoleBindingBuilder{
		Instance: builder.Instance,
		Scheme:   builder.Scheme,
	}
}

func (builder *AnyuidRoleBindingBuilder) UpdateMayRequireStsRecreate() bool {
	return false
}

func (builder *AnyuidRoleBindingBuilder) Update(object client.Object) error {
	roleBinding := object.(*rbacv1.RoleBinding)
	roleBinding.Labels = metadata.GetLabels(builder.Instance.Name, builder.Instance.Labels)
	roleBinding.Annotations = metadata.ReconcileAndFilterAnnotations(roleBinding.GetAnnotations(), builder.Instance.Annotations)
	roleBinding.RoleRef = rbacv1.RoleRef{
		APIGroup: "rbac.authorization.k8s.io",
		Kind:     "ClusterRole",
		Name:     "system:openshift:scc:anyuid",
	}
	roleBinding.Subjects = []rbacv1.Subject{
		{
			Kind: "ServiceAccount",
			Name: builder.Instance.ChildResourceName(serviceAccountName),
		},
	}

	if err := controllerutil.SetControllerReference(builder.Instance, roleBinding, builder.Scheme); err != nil {
		return fmt.Errorf("failed setting controller reference: %v", err)
	}
	return nil
}

func (builder *AnyuidRoleBindingBuilder) Build() (client.Object, error) {
	return &rbacv1.RoleBinding{
		ObjectMeta: metav1.ObjectMeta{
			Namespace: builder.Instance.Namespace,
			Name:      builder.Instance.ChildResourceName(anyuidRoleBindingName),
		},
	}, nil
}
```
3. 在 rabbitmq-operator/internal/resource/rabbitmq_resource_builder.go 文件第 39 行下添加以下代码
```
builder.AnyuidRoleBinding(),
```
4. 修改 rabbitmq-operator/Dockerfile 第 9 行，修改代理，解决国内下载依赖网络问题
```shell script
RUN go env -w GOPROXY=https://goproxy.cn \
    && go mod download
```
5. 将 rabbitmq operator controller 重新构建上传至镜像仓库
```shell script
docker build -t harbor.test.geely.com/operator/rabbitmq-operator-controller:base .
docker push harbor.test.geely.com/operator/rabbitmq-operator-controller:base
```

#### 使用命令行工具生产文件 
1. 生成 csv manifests，按照提示输入 operator 描述信息和提供者信息 
```shell script
operator-sdk generate kustomize manifests
```
2. 修改 rabbitmq-operator/config/manager/kustomization.yaml 文件最后两行，修改为上面 controller 的镜像和 tag 
```yaml
 newName: harbor.test.geely.com/operator/rabbitmq-operator-controller
 newTag: base
```
3. 修改 rabbitmq-operator/config/manifests/kustomization.yaml 为
```yaml
resources:
- ../samples/base
- ../installation
```
4. 修改 rabbitmq-operator/config/manifests/bases/rabbitmq-operator.clusterserviceversion.yaml
```yaml
# 第 15 行修改为 icon 文件 base64 encode 的值，第 16 行为 icon 文件格式
- base64data: "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"
  mediatype: "image/png"
# 第 22，24，26，28 行修改为 true 
- supported: true
```
5. 生成 csv 文件，会放在 rabbitmq-operator/bundle 目录下，bundle.Dockerfile 文件放在 rabbitmq-operator 目录下
```shell script
kustomize build config/manifests | operator-sdk generate bundle --package rabbitmq-operator --version 1.6.0
```
6. 验证 csv 文件
```shell script
operator-sdk bundle validate ./bundle
```
7. 打包上传 csv 文件镜像到镜像仓库
```shell script
docker build -f bundle.Dockerfile -t harbor.test.geely.com/operator/rabbitmq-operator:v1.6.0 .
docker push harbor.test.geely.com/operator/rabbitmq-operator:v1.6.0
```
8. 检查编译的镜像是正确
```shell script
opm alpha bundle validate --tag harbor.test.geely.com/operator/rabbitmq-operator:v1.6.0 --image-builder docker
```
9. 使用opm 命令将csv 镜像内容打包到 opm index 镜像中，并上传到镜像仓库
```shell script
opm index add -b harbor.test.geely.com/operator/rabbitmq-operator:v2.6.0 -t harbor.test.geely.com/operator/geely-index:v1 -u docker
docker push harbor.test.geely.com/operator/geely-index:v1
```
10. 创建自定义 catalog 源
```shell script
cat << EOF | oc create -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: geely-catalog
  namespace: openshift-marketplace
spec:
  displayName: geely-catalog
  image: harbor.test.geely.com/operator/geely-index:v1
  publisher: geely
  sourceType: grpc 
EOF
```
11. 安装 operator 到 openshift-operators namespace 下，并对 sa rabbitmq-cluster-operator 添加 scc anyuid 权限
```shell script
oc adm policy add-scc-to-user anyuid -z rabbitmq-cluster-operator
```
