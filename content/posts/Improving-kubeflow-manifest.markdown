+++
title = "Improving Kubeflow kfctl_istio_dex.v1.0.2 manifest"
description = ""
tags = [
    "Kubernetes",
    "Kubeflow",
]
date = "2020-06-17"
categories = [
    "Kubernetes",
    "MLOps",
    "Kubeflow",
]
highlight = "true"
+++

Kubeflow가 버전 1.0을 찍긴 했지만 아직은 부족한 부분이 좀(?) 있다.
kfctl과 KFDef를 분석할 겸해서 kfctl_istio_dex.v1.0.2 기반으로 지금까지 customizing했던 부분들을 적용하여 kubeflow/manifests 를 개선해 보았다.

### 개선한 사항

- Pipeline 0.5.1 로 업그레이드
- KFServing 0.3 으로 업그레이드
- kfserving-ingressgateway 적용 (https://github.com/kubeflow/kfserving/issues/786 로 인한 Workaround)
- nodeafinity 적용(kubeflow 배포 철학과는  맞지 않지만....)
- Application CR 빠진 부분 추가
- 중복 지정 부분(namespace 등) 삭제
- minio vitualservice 추가

Git repo - https://github.com/jazzsir/manifests/tree/v1.0.2-branch

### 설치 방법

설치 방법은 기존과 같다. manifest 파일만 수정한 파일로 지정하시면 된다.

- 일반 설치

```
kfctl apply -V -f https://raw.githubusercontent.com/jazzsir/manifests/v1.0.2-branch/kfdef/kfctl_istio_dex.v1.0.2.jazzsir.yaml
```

- nodeaffinity(kubernetes.io/nodetype:kubeflow) 적용

```
kfctl apply -V -f https://raw.githubusercontent.com/jazzsir/manifests/v1.0.2-branch/kfdef/kfctl_istio_dex.v1.0.2.nodeaffinity.yaml
```

### 테스트 항목
Fairing, Katib, tfjob(GPU활용), mpijob(GPU활용), Metadata, KFServing, Pipeline(PVC/S3 적용, Katib/tfjob/KFServing/web_ui 등 포함),  사용자 인증 등

### 마무리
기존에 정리한게 있어서 몇 시간이면 될 줄 알았는데, 컴포넌트간에 중복 항목이 많고, kfctl build도 가능하게 하려다보니 kustomize overlay사용에 kfctl 제약사항이 많아서 예상보다 오래 걸렸다.
