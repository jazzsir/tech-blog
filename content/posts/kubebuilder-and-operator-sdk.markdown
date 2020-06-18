+++
title = "Kubebuilder and Operator-SDK Souce code 분석"
description = ""
tags = [
    "Operator",
    "Controller",
    "Operator-SDK",
    "Kubebuilder",
]
date = "2019-06-12"
categories = [
    "Kubernetes",
]
highlight = "true"
+++
## Kubernetes Operator/Controller 구현 방법

Kubernetes에서 Operator(또는 controller)를 구현 하려면 여러가지 방법이 있다.
1. API Server의 REST API(watch)를 이용하여 shell script로 구현: 가장 간단한 방법이나 오류 처리나 고급 기능을 이용하는데 한계가 있다.
2. client-go library를 직접 이용: workqueue 구현이나 client, informers, listers, deep-copy 등의 기능들을 개발자 영역에서 모두 구현해야 하지만, 이러한 코드들을 자동으로 생성해주는 방법이 있으므로 사실 그렇게 어렵지는 않다.
3. operator-sdk 또는 kubebuilder와 같은 SDK를 이용하는 방법: controller나 operator를 개발할 때, 가장 많이 사용하고 있는 방법이다.
개발자가 API와 controller 이름을 지정해서 생성하면 기본적인 scaffold를 자동으로 생성해 줘서 개발자가 코딩해야 할 부분이 많이 줄어들었다.
뿐만 아니라, client-go기반의 controller는 많은 기능들이 개발자 영역에서 구현 해야 했지만 controller-runtime같은 libraries로 개발자 영역에서 구현해야 할 기능들이 대폭 줄어 들었다.
4. Metacontroller를 이용하여 구현: Custom Controller를 만드는 공통되는 부분을 캡슐화하는 API를 이용하여 kubernetes를 확장한다.

여기에서는 현재 가장 많이 사용되고 있는 Kuberbuilder와 Operator-SDK에 대해서 살펴본다.

## controller-runtime
 kubebuilder와 Operator-sdk 모두 controller-runtime기반인다. 그래서 소스코드 호환도 가능하다.
SDK를 설명하기전에 이것부터 간략하게 살펴보자
- 가장 핵심이 되는 부분은 아래 3가지 이다.
  - client.Client : k8s cluster에 CRUD 실행
  - manager.Manager : shared dependencies 관리 예를 들어 cache, client 등
  - reconcile.Reconcile : 현재 상태와 제공된 상태 비교하고 update 한다
  좀더 엄밀히 말하면 controller내에 개발자가 구현하는 Reconcile()은 reconcile.Reconcile() interface를 구현하는 것이고,
  imformer에 등록된 GVK에 변경이 일어나서 callback handler가 호출되고 여기서 Reconcile() method를 호출되고 여기서 client가 update하거나 사용자가 정의한 기능이 실행된다.</br>
그리고 clientset으로 해당 자원만 watching하는 informer를 사용하면 server에 부담을 줄 수 있다.
그래서 cache를 이용해서 sharedinformer로 여러 resources를 watching 한다.

## Kubebuilder

Kubebuilder는 Operator-sdk와 비교해서 더 많은 기능들이 있다.
1. webhook을 지원한다. webhook도 결국엔 webserver이므로 kubebuilder로 쉽게 구현이 가능하다.
2. Operator-sdk는 GVK가 하나의 controller만 사용할 수 있었는데, Kubebuilder는 이런 제약이 없다.
3. Kustomize를 이용해서 한 방에 deploy가 가능하다.

### Version 2.x

Version 2.x는 1.x와 비교해서 많이 바뀌었다.
1. 디렉토리 구조가 한 단계씩 올라왔다.
	- 이전엔 $WORKSPACE/pkg/controller, $WORKSPACE/pkg/api, $WORKSPACE/cmd/main.go 였는데
 $WORKSPACE/controller, $WORKSPACE/api, $WORKSPACE/main.go로 한 단계씩 올라왔다.
2. 이전 버전은 controller의 Add()를 호출해서 reconcile을 등록했는데
이제는 main.go 에서 controller.StarshipReconciler를 호출하고.. 호출하고.. 해서 아래로 등록됨
	- ctrl.NewControllerManagedBy(mgr).For(&mygroupv1beta1.Mykind{}).Complete(r)
3. main.go 에서 manager 등록, scheme 등록, webHook 등록 등이 나눠져 있었는데 reconciler 등록 할때 모두 한꺼번에 다 한다.

### 소스코드 흐름(v2.x)

- main.go()
	- init()
		- webappv1.AddToScheme(scheme)
			- api/v1/guestbook_types.go에서 아래로 등록한 scheme이 add됨.
				- SchemeBuilder.Register(&Guestbook{}, &GuestbookList{})

	- mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		- GetConfigOrDie()
			- .kube/config 파일 참조함
		- /sigs.k8s.io/controller-runtime/pkg/manager/manager.go
			- New()
				- 예전 버전에서는 이부분을 inject라고 했는데 새로운 버전에서는 inject라는 용어는 사용안하는것 같음
				- options = setOptionDefaults
					- Client, Scheme, leadelection, metrics관련, cacheFunc(informerMap관련 함수들)등등을 설정할 수 있는 함수를 등록
				- cache, err := options.NewCache
					- setOptionDefaults에서 지정한 함수로 Cache 생성
					- controller-runtime/pkg/cache/cache.go
						- New(config....)
							- GVK Group version Kind 별로 informer map을 만들어서 cache를 생성
								- 이렇게 되면 GVK별로 informer가 처리되고 Shared dependencies 즉, cache가 생성됨
				- apiReader, err := client.New
					- API Server에 read할 client 생성함
				- writeObj, err := option.NewClient
					- API Server에 write할 client 생성함
				- resourceLock, err := options.newResourceLock
					- Leader election을 위한 resourceLock 생성
				- return &controllerManager
	- err = ($controllers.MykindReconciler{Client: mgr.GetClient(), Log:...}).SetupWithManager(mgr)
		- controller에 reconciler를 등록함
		- mgr.GetClient()는 Apiserver에 write하기 위한 Client
		- controllers/mykind_controller.go
			- SetupWithManager(mgr) -> ctrl.NewControllerManagedBy(mgr).For(&mygroupv1beta1.Mykind{}).Complete(r)
				- (controller-runtime/pkg/builder/build.go)
				- NewControllerManagedBy(mgr): controller를 build하기 위해 `builder` package를 호출
				- For(&mygroupv1beta1.Mykind{}): mykind를 builder에 등록
				- 즉, Controller를 실제적으로 만드는 곳임
				- Complete(r)
					- controller에 작성한 business logic 코드를 넣은 Reconciler를 등록함.
					- Build()
						- blder.doConfig() : set the Config
							- controller에 manager의 config를 설정하는것 같다.
						- blder.doManager() : set the Manager
						- blder.doController() : set the ControllermanagedBy
							- 실제 manager에 controller를 등록함. 여기서 controller에 여러분이 작성한 business logic인 Reconciler가 등록
						- blder.doWebhook() : set the Webhook if needed
							- 위의 `For`에서 등록한 GVK(Group, V?, Kind)인 mykind를 이용하여
							- admission의 Mutating Webhook과 Validating Webhook을 이용 가능한듯함.
						- blder.doWatch() : set the Watch
							- `For`에서 등록한 API type(mykind)를 watch할 수 있도록 work queue에 call back 함수인 handler를 등록함.
							- 예전 버전과 가장 큰 차이중에 하나입니다. 이전엔 contoller에서 Add를 구현해서 직접 watch 함수로 Watch할 GVK를 등록했었음, 근데 새로운 버전에서는 자동으로 지정함.
							- doWatch()에 들어가 보면
								- 등록한 GVK가 자동으로 등록됨.
								- blder.managedObjects 를 찾아보면 Owns 로 등록되는데 이건 handler가 기본 파리미터러 등록되어 있음.
								- blder.watchRequest 를 찾아보면 Watches로 등록되는데 이건 handler를 직접 지정할 수 있음.

	- if err := mgr.Start(ctrl.SetupSignalHandler())
		- controller-runtime/pkg/internal/manager/internal.go
			- Start()
				- startNonLeaderElectionRunnables()
					- controller-runtime/pkg/internal/controller/controller.go
						- Start()
							- 여기에서 실제적으로 work queue를 만들고 hander를 등록해서 queue에 해당 GVK에
							해당하는 object가 들어오면(delete/update/add) reconciler에 전달할 수 있게 설정하는 곳.
							- c.WaitForCacheSync(stop) - 반복문 돌면서 informerMap의 informer들이 API - Server와 sync될때까지 기다린다.
							- go wait.Until(c.worker, c.JitterPeriod, stop)
								- func (c *Controller) worker() {
									- func (c *Controller) processNextWorkItem() bool {
										- 왜 NextWork라고 했냐하면 Work가 여러개 일 수가 있기 때문입니다. 다시 말해서 하나의 controller에서 다수의 Reconcile()함수가 running될 수 있음.
										그래서 각 하나의 reconcile한수당 하나의 work라고 생각하시면 됨.
										- func (c *Controller) reconcileHandler(obj interface{}) bool {
											- 이제 이 변경이 되면 여기 handler가 호출됨.
											- if req, ok = obj.(reconcile.Request); !ok {
												- queue에서 request를 가져와서
											- if result, err := c.Do.Reconcile(req); err != nil {
												- 개발자가 만든 business logic이 있는 Reconcile()함수를 호출함.
												- c.Queue.Forget(obj)
													- 그리고 work queue에서 해당 object를 삭제.


## Operator-SDK

### 소스코드 흐름

- main.go()
	- leader.Become: operator를 HA로 줄 수도 있는데 하나의 operator만 운영될 수 있도록 설정하는 부분임. 이게 실행되면 configmap에 leader가 된 operator pod name이 설정된다.
		- operator-framework/operator-sdk/pkg/leader/leader.go
			- 기존에 operator가 있다면 에러 리턴
			- 없으면 configmap에 등록하고 실행
	- manager.New(cfg, manager.Options{
		- controller-runtime/pkg/manager/manager.go
			- options = setOptionDefaults
				- option에 기본값 설정 - option(namespace, leadelection(bool), metricsBindAddress, cacheFunc, Client, Scheme(이건 default로 runtime.Scheme 이용) )으로 client를 만든다
			- cache, err := options.NewCache
				- controller-runtime/pkg/cache/cache.go
					- New(config....)
						- GVK Group version Kind 별로 informer map을 만들어서 cache를 생성
							- 이렇게 되면 GVK별로 informer가 처리되는것 같다.
							- 이렇게 해서 Shared dependencies 즉, cache를 생성하고. manager에 등록된다.
	- apis.AddToScheme
		- pkg/apis/apis.go
			- apis.AddToScheme을 통해서 register.go로 통해 등록된 scheme이 등록된다.
	- controller.AddToManager(mgr)
		- apis.controller.add_memcached.go
			- init()에서 등록한 controller를 등록함.
			- f(m) = Add(mgr manager.Manager) 를 실행해서 controller 아래에 있는 모든 controller들을 등록함.
		- apis.controller.memcached.memcached_controller.go
			- Add(mgr manager.Manager)
				- newReconciler(mgr)
					- ReconcileMemcached 는 controller에 등록된 reconcile.Reconciler 인터페이스이다.
					  그래서 아래의 ReconcileMemcached 구초체 함수인 Reconcile 함수가 자동적으로 controller에서 호출됨.
				- add(mgr, newReconciler(mgr))
					- controller.New
						- controller-runtime/pkg/controller/controller.go
							- SetFields()
								- options.Reconciler에 이전에 manager에 등록한 options들을 copy함.
								- 그리고 workqueue가 등록된 controller를 리턴함.
					- c.Watch......cachev1alpha1
						- controller-runtime/pkg/internal/controller/controller.go
							- Watch()
								- Controller에 kind와 handler를 등록하고
								- controller-runtime/pkg/internal/source/source.go
									- kind에 해당하는 informer를 찾아서(없으면 새로 등록합니다. GVK 즉, Group version Kind 별로 informer map을 관리하고 있음)
									  handler(add/delete/update), queue를 등록해서 watch를 실행함.
	- mgr.Start(signals.SetupSignalHandler()) - SIGTERM , SIGINIT 등을 받아들이기 위함
		- controller-runtime/pkg/internal/manager/internal.go
			- Start()
				- start()
					- cm.cache.WaitForCacheSync(cm.internalStop)
					  - 반복문 돌면서 informerMap의 informer들이 API Server와 sync될때까지 기다린다.
						- controller-runtime/pkg/internal/controller/controller.go
							- Start()
								- 여기서 변경이 일어나면,
								- for c.processNextWorkItem() 으로 반복문 돌면서
									- if result, err := c.Do.Reconcile(req); 에서 변경이 일어나면 Reconcile을 호출함.

